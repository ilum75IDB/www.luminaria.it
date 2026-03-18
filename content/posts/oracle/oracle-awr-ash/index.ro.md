---
title: "AWR, ASH și cele 10 minute care au salvat un go-live"
description: "Vineri seara, în ajunul unui go-live. Performanța se prăbușește. Cu AWR și ASH am găsit un full table scan ascuns într-o procedură stocată în mai puțin de zece minute — iar lansarea în producție a mers înainte."
date: "2026-02-10T10:00:00+01:00"
draft: false
translationKey: "oracle_awr_ash"
tags: ["awr", "ash", "performance", "tuning", "go-live", "diagnostic"]
categories: ["oracle"]
image: "oracle-awr-ash.cover.jpg"
---

Vineri, ora 18:40. Aveam deja geaca pe mine, gata de plecare. Telefonul vibrează. E project managerul.

"Ivan, avem o problemă. Sistemul merge foarte lent. Mâine dimineață e go-live-ul."

Nu e prima dată când primesc un astfel de apel. Dar tonul era diferit. Nu era plângerea obișnuită despre lentoare. Era panică.

Mă reconectez prin VPN, deschid o sesiune pe baza de date Oracle 19c a clientului. Primul lucru pe care îl fac e o verificare rapidă:

``` sql
SELECT metric_name, value
FROM   v$sysmetric
WHERE  metric_name IN ('Database CPU Time Ratio',
                       'Database Wait Time Ratio',
                       'Average Active Sessions');
```

**CPU Time Ratio**: 12%. În condiții normale era peste 80%.

**Average Active Sessions**: 47. Pe un server cu 16 core-uri.

Patruzeci și șapte de sesiuni active. Baza de date se îneca.

------------------------------------------------------------------------

## 🔥 Simptomele

Echipa de dezvoltare terminase ultimul deploy al codului aplicativ în acea după-amiază. Totul părea să funcționeze pe mediul de test. Dar când au lansat batch-ul de verificare pre-go-live — cel care simulează sarcina de producție — timpii de răspuns au explodat.

Query-urile care în mod normal rulau în 2-3 secunde durau 45. Batch-urile care terminau în 20 de minute erau încă în execuție după o oră. {{< glossary term="wait-event" >}}Wait event{{< /glossary >}}-urile dominante erau `db file sequential read` și `db file scattered read` — semn clar de I/O fizic masiv.

Ceva citea cantități enorme de date de pe disc. Ceva care înainte nu era acolo.

------------------------------------------------------------------------

## 📊 AWR: fotografia problemei

{{< glossary term="awr" >}}AWR{{< /glossary >}} — Automatic Workload Repository — este cel mai puternic instrument de diagnostic pe care Oracle îl pune la dispoziție. În fiecare oră, Oracle face o captură ({{< glossary term="snapshot-oracle" >}}snapshot{{< /glossary >}}) a statisticilor de performanță și o stochează în repository-ul intern. Comparând două snapshot-uri, obții un raport care îți spune exact ce s-a întâmplat în acea perioadă.

Am generat un snapshot manual pentru a captura situația curentă:

``` sql
EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;
```

Apoi am căutat snapshot-urile disponibile:

``` sql
SELECT snap_id, begin_interval_time, end_interval_time
FROM   dba_hist_snapshot
WHERE  begin_interval_time > SYSDATE - 1/6
ORDER BY snap_id DESC;
```

Aveam un snapshot de la 18:00 (înainte de problema vizibilă) și cel tocmai creat la 18:45. Am generat raportul AWR:

``` sql
SELECT output
FROM   TABLE(DBMS_WORKLOAD_REPOSITORY.awr_report_text(
         l_dbid     => (SELECT dbid FROM v$database),
         l_inst_num => 1,
         l_bid      => 4523,
         l_eid      => 4524
       ));
```

### Ce spunea raportul

Secțiunea **Top 5 Timed Foreground Events** era grăitoare:

| Event | Waits | Time (s) | % DB time |
|---|---|---|---|
| db file scattered read | 1.247.832 | 3.847 | 58,2% |
| db file sequential read | 423.109 | 1.205 | 18,2% |
| CPU + Wait for CPU | — | 892 | 13,5% |
| log file sync | 12.445 | 287 | 4,3% |
| direct path read | 8.221 | 198 | 3,0% |

`db file scattered read` la 58%. Sunt {{< glossary term="full-table-scan" >}}full table scan{{< /glossary >}}-uri. Ceva citea tabele întregi, bloc cu bloc, fără a folosi indecși.

Secțiunea **SQL ordered by Elapsed Time** arăta un singur SQL_ID care consuma 71% din timpul total al bazei de date: `g4f2h8k1nw3z9`.

Acum știam ce să caut.

------------------------------------------------------------------------

## 🔍 ASH: microscopul

AWR îmi dăduse fotografia de ansamblu. Dar trebuia să înțeleg **când** a început acel SQL, **cine** îl executa și **ce program** l-a lansat.

{{< glossary term="ash" >}}ASH{{< /glossary >}} — Active Session History — înregistrează starea fiecărei sesiuni active o dată pe secundă. Este microscopul DBA-ului: unde AWR îți arată medii pe o oră, ASH îți arată ce se întâmpla secundă cu secundă.

``` sql
SELECT sample_time,
       session_id,
       sql_id,
       sql_plan_hash_value,
       event,
       program,
       module
FROM   v$active_session_history
WHERE  sql_id = 'g4f2h8k1nw3z9'
  AND  sample_time > SYSDATE - 1/24
ORDER BY sample_time DESC;
```

Rezultatele erau clare:

- **Program**: `JDBC Thin Client` — aplicația Java a batch-ului
- **Module**: `BatchVerificaProduzione`
- **Event**: `db file scattered read` în 92% din eșantioane
- **Prima apariție**: 17:12 — exact după deploy-ul din după-amiază
- **SQL_PLAN_HASH_VALUE**: `2891047563`

{{< glossary term="execution-plan" >}}Planul de execuție{{< /glossary >}} se schimbase. Înainte de deploy, acea query folosea un plan diferit.

------------------------------------------------------------------------

## 🧩 Planul de execuție

Am recuperat planul curent:

``` sql
SELECT *
FROM   TABLE(DBMS_XPLAN.display_awr(
         sql_id          => 'g4f2h8k1nw3z9',
         plan_hash_value => 2891047563
       ));
```

Rezultatul mi-a făcut problema evidentă imediat:

```
---------------------------------------------------------------------------
| Id | Operation            | Name            | Rows  | Cost  |
---------------------------------------------------------------------------
|  0 | SELECT STATEMENT     |                 |       | 48721 |
|  1 |  HASH JOIN           |                 | 2.1M  | 48721 |
|  2 |   TABLE ACCESS FULL  | MOVIMENTI_TEMP  | 2.1M  | 41893 |
|  3 |   INDEX RANGE SCAN   | IDX_CLIENTI_PK  |     1 |     2 |
---------------------------------------------------------------------------
```

**TABLE ACCESS FULL pe MOVIMENTI_TEMP**. O tabelă temporară cu 2,1 milioane de rânduri, citită integral de fiecare dată. Fără index. Fără filtru eficient.

Am verificat ce exista înainte de deploy consultând planul anterior în AWR:

``` sql
SELECT plan_hash_value, timestamp
FROM   dba_hist_sql_plan
WHERE  sql_id = 'g4f2h8k1nw3z9'
ORDER BY timestamp;
```

Planul anterior (hash `1384726091`) folosea un `INDEX RANGE SCAN` pe un index care — descoperire — **fusese eliminat în timpul deploy-ului**. Scriptul de migrare includea un `DROP TABLE MOVIMENTI_TEMP` urmat de o recreare, dar fără a recrea indexul.

------------------------------------------------------------------------

## ⚡ Soluția

Zece minute. Din momentul conectării până la identificarea cauzei. Nu din pricepere — din cauza instrumentelor.

Fix-ul era simplu:

``` sql
CREATE INDEX idx_movimenti_temp_cliente
ON movimenti_temp (id_cliente, data_movimento)
TABLESPACE idx_data;
```

După crearea indexului, am forțat un re-parse al query-ului:

``` sql
EXEC DBMS_SHARED_POOL.purge('g4f2h8k1nw3z9', 'C');
```

Am cerut echipei să relanseze batch-ul. Timp de execuție: 18 minute. Identic cu testele anterioare.

Go-live-ul de sâmbătă dimineață a decurs normal.

------------------------------------------------------------------------

## 📋 AWR vs ASH: când să folosești ce

După acel episod am formalizat o regulă pe care o urmez întotdeauna:

| Caracteristică | AWR | ASH |
|---|---|---|
| Granularitate | Snapshot-uri orare (configurabile) | Eșantion în fiecare secundă |
| Adâncime istorică | Până la 30 zile (implicit 8) | 1 oră în memorie, apoi în AWR |
| Caz de utilizare principal | Analiză de tendințe, comparare perioade | Diagnostic punctual, izolare SQL |
| View principal | `DBA_HIST_*` | `V$ACTIVE_SESSION_HISTORY` |
| View istoric | — | `DBA_HIST_ACTIVE_SESS_HISTORY` |
| Licență necesară | Diagnostic Pack | Diagnostic Pack |
| Output tipic | Raport HTML/text | Query-uri ad hoc |

Regula empirică: **AWR ca să înțelegi ce s-a schimbat, ASH ca să înțelegi de ce**.

AWR îți spune: "Între 17:00 și 18:00, 58% din timpul bazei de date a fost petrecut pe full table scan-uri." ASH îți spune: "La 17:12:34, sesiunea 847 executa query-ul g4f2h8k1nw3z9 cu un full table scan pe MOVIMENTI_TEMP, lansat de programul BatchVerificaProduzione."

Sunt complementare. Să folosești doar unul e ca și cum ai diagnostica o problemă uitându-te doar la CT sau doar la analizele de sânge.

------------------------------------------------------------------------

## 🛡️ Query-urile pe care orice DBA ar trebui să le aibă pregătite

De-a lungul anilor am construit un set de query-uri de diagnostic pe care le am mereu la îndemână. Le împărtășesc pentru că într-o urgență nu ai timp să le scrii de la zero.

### Top SQL după timp de execuție (ultima oră)

``` sql
SELECT sql_id,
       COUNT(*) AS samples,
       ROUND(COUNT(*) / 60, 1) AS est_minutes,
       MAX(event) AS top_event,
       MAX(program) AS program
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/24
  AND  sql_id IS NOT NULL
GROUP BY sql_id
ORDER BY samples DESC
FETCH FIRST 10 ROWS ONLY;
```

### Distribuția wait event-urilor pentru un SQL specific

``` sql
SELECT event,
       COUNT(*) AS samples,
       ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER(), 1) AS pct
FROM   v$active_session_history
WHERE  sql_id = '&sql_id'
  AND  sample_time > SYSDATE - 1/24
GROUP BY event
ORDER BY samples DESC;
```

### Compararea planurilor de execuție în timp

``` sql
SELECT plan_hash_value,
       MIN(timestamp) AS first_seen,
       MAX(timestamp) AS last_seen,
       COUNT(*) AS executions_in_awr
FROM   dba_hist_sqlstat
WHERE  sql_id = '&sql_id'
GROUP BY plan_hash_value
ORDER BY first_seen;
```

------------------------------------------------------------------------

## 🎯 Ce am învățat în acea seară

Trei lecții pe care le port cu mine.

**Prima**: deploy-ul nu este doar cod. Este și structură. Când lansezi în producție, trebuie să verifici că indecșii, constraint-urile, statisticile și grant-urile sunt coerente cu ce era acolo înainte. Un script care face `DROP TABLE` și `CREATE TABLE` fără a recrea indecșii e o bombă cu ceas.

**A doua**: AWR și ASH nu sunt instrumente pentru DBA seniori. Sunt instrumente de primă linie, ca un defibrilator. Trebuie să știi să le folosești înainte de a avea nevoie de ele, nu în timpul urgenței.

**A treia**: zece minute de diagnostic corect valorează mai mult decât trei ore de încercări oarbe. Când sistemul e în genunchi, tentația e să repornești, să omori sesiuni, să adaugi resurse. Dar fără să știi ce se întâmplă, tragi în întuneric.

În acea seară am plecat de la birou la 19:20. La patruzeci de minute de la apelul telefonic. A doua zi go-live-ul a pornit fără probleme, iar luni sistemul mergea normal.

Nu sunt un erou. Am folosit doar instrumentele potrivite.

------------------------------------------------------------------------

## Glosar

**[AWR](/ro/glossary/awr/)** — Automatic Workload Repository. Componenta integrata in Oracle care colecteaza statistici de performanta prin snapshot-uri periodice si genereaza rapoarte de diagnostic comparative.

**[ASH](/ro/glossary/ash/)** — Active Session History. Componenta Oracle care esantioneaza starea fiecarei sesiuni active o data pe secunda, stocand-o in memorie si apoi in AWR. Este microscopul DBA-ului pentru diagnosticarea punctuala.

**[Full Table Scan](/ro/glossary/full-table-scan/)** — Operatie de citire in care Oracle citeste toate blocurile unei tabele fara a folosi indecsi. In wait event-uri apare ca `db file scattered read`.

**[Wait Event](/ro/glossary/wait-event/)** — Eveniment de asteptare inregistrat de Oracle de fiecare data cand o sesiune nu poate continua pentru ca asteapta o resursa (I/O, lock, CPU, retea). Analiza wait event-urilor este baza metodologiei de diagnostic Oracle.

**[Snapshot](/ro/glossary/snapshot-oracle/)** — Captura punctuala a statisticilor de performanta preluata periodic de AWR (implicit la fiecare 60 de minute). Compararea a doua snapshot-uri genereaza raportul AWR.
