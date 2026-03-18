---
title: "AWR, ASH e i 10 minuti che hanno salvato un go-live"
description: "Venerdì sera, vigilia di un go-live. Le performance crollano. Con AWR e ASH ho trovato un full table scan nascosto in una stored procedure in meno di dieci minuti — e il rilascio in produzione è andato avanti."
date: "2026-02-10T10:00:00+01:00"
draft: false
translationKey: "oracle_awr_ash"
tags: ["awr", "ash", "performance", "tuning", "go-live", "diagnostic"]
categories: ["oracle"]
image: "oracle-awr-ash.cover.jpg"
---

Venerdì, ore 18:40. Ero già con il giubbotto addosso, pronto per uscire. Il telefono vibra. È il project manager.

"Ivan, abbiamo un problema. Il sistema è lentissimo. Domani mattina c'è il go-live."

Non è la prima volta che ricevo una chiamata del genere. Ma il tono era diverso dal solito. Non era la lamentela generica sulla lentezza. Era panico.

Mi ricollego in VPN, apro una sessione sul database Oracle 19c del cliente. La prima cosa che faccio è un controllo rapido:

``` sql
SELECT metric_name, value
FROM   v$sysmetric
WHERE  metric_name IN ('Database CPU Time Ratio',
                       'Database Wait Time Ratio',
                       'Average Active Sessions');
```

**CPU Time Ratio**: 12%. In condizioni normali era sopra l'80%.

**Average Active Sessions**: 47. Su un server con 16 core.

Quarantasette sessioni attive. Il database stava annegando.

------------------------------------------------------------------------

## 🔥 I sintomi

Il team di sviluppo aveva completato l'ultimo deploy del codice applicativo quel pomeriggio. Tutto sembrava funzionare sui test. Ma quando hanno lanciato il batch di verifica pre-go-live — quello che simula il carico di produzione — i tempi di risposta sono esplosi.

Le query che normalmente giravano in 2-3 secondi ne impiegavano 45. I batch che finivano in 20 minuti erano ancora in esecuzione dopo un'ora. I {{< glossary term="wait-event" >}}wait event{{< /glossary >}} dominanti erano `db file sequential read` e `db file scattered read` — segno inequivocabile di I/O fisico massiccio.

Qualcosa stava leggendo enormi quantità di dati dal disco. Qualcosa che prima non c'era.

------------------------------------------------------------------------

## 📊 AWR: la fotografia del problema

{{< glossary term="awr" >}}AWR{{< /glossary >}} — Automatic Workload Repository — è lo strumento diagnostico più potente che Oracle mette a disposizione. Ogni ora, Oracle scatta un'istantanea ({{< glossary term="snapshot-oracle" >}}snapshot{{< /glossary >}}) delle statistiche di performance e la conserva nel repository interno. Confrontando due snapshot, ottieni un report che ti dice esattamente cosa è successo in quel periodo.

Ho generato uno snapshot manuale per catturare la situazione attuale:

``` sql
EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;
```

Poi ho cercato gli snapshot disponibili:

``` sql
SELECT snap_id, begin_interval_time, end_interval_time
FROM   dba_hist_snapshot
WHERE  begin_interval_time > SYSDATE - 1/6
ORDER BY snap_id DESC;
```

Avevo uno snapshot delle 18:00 (prima del problema visibile) e quello appena creato alle 18:45. Ho generato il report AWR:

``` sql
SELECT output
FROM   TABLE(DBMS_WORKLOAD_REPOSITORY.awr_report_text(
         l_dbid     => (SELECT dbid FROM v$database),
         l_inst_num => 1,
         l_bid      => 4523,   -- snapshot inizio
         l_eid      => 4524    -- snapshot fine
       ));
```

### Cosa diceva il report

La sezione **Top 5 Timed Foreground Events** era eloquente:

| Event | Waits | Time (s) | % DB time |
|---|---|---|---|
| db file scattered read | 1.247.832 | 3.847 | 58,2% |
| db file sequential read | 423.109 | 1.205 | 18,2% |
| CPU + Wait for CPU | — | 892 | 13,5% |
| log file sync | 12.445 | 287 | 4,3% |
| direct path read | 8.221 | 198 | 3,0% |

`db file scattered read` al 58%. Sono {{< glossary term="full-table-scan" >}}full table scan{{< /glossary >}}. Qualcuno stava leggendo tabelle intere, blocco dopo blocco, senza usare indici.

La sezione **SQL ordered by Elapsed Time** mostrava un solo SQL_ID che consumava il 71% del tempo totale del database: `g4f2h8k1nw3z9`.

Ora sapevo cosa cercare.

------------------------------------------------------------------------

## 🔍 ASH: il microscopio

AWR mi aveva dato la fotografia d'insieme. Ma avevo bisogno di capire **quando** quel SQL era iniziato, **chi** lo eseguiva, e **quale programma** lo aveva lanciato.

{{< glossary term="ash" >}}ASH{{< /glossary >}} — Active Session History — registra lo stato di ogni sessione attiva una volta al secondo. È il microscopio del DBA: dove AWR ti mostra le medie su un'ora, ASH ti mostra cosa succedeva secondo per secondo.

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

I risultati erano chiari:

- **Program**: `JDBC Thin Client` — l'applicazione Java del batch
- **Module**: `BatchVerificaProduzione`
- **Event**: `db file scattered read` nel 92% dei campioni
- **Prima occorrenza**: 17:12 — esattamente dopo il deploy del pomeriggio
- **SQL_PLAN_HASH_VALUE**: `2891047563`

Il {{< glossary term="execution-plan" >}}piano di esecuzione{{< /glossary >}} era cambiato. Prima del deploy, quella query usava un piano diverso.

------------------------------------------------------------------------

## 🧩 Il piano di esecuzione

Ho recuperato il piano corrente:

``` sql
SELECT *
FROM   TABLE(DBMS_XPLAN.display_awr(
         sql_id          => 'g4f2h8k1nw3z9',
         plan_hash_value => 2891047563
       ));
```

Il risultato mi ha fatto capire subito il problema:

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

**TABLE ACCESS FULL su MOVIMENTI_TEMP**. Una tabella temporanea con 2,1 milioni di righe, letta per intero ogni volta. Nessun indice. Nessun filtro efficace.

Ho verificato cosa esisteva prima del deploy controllando il piano precedente in AWR:

``` sql
SELECT plan_hash_value, timestamp
FROM   dba_hist_sql_plan
WHERE  sql_id = 'g4f2h8k1nw3z9'
ORDER BY timestamp;
```

Il piano precedente (hash `1384726091`) usava un `INDEX RANGE SCAN` su un indice che — scoperta — **era stato eliminato durante il deploy**. Lo script di migrazione includeva un `DROP TABLE MOVIMENTI_TEMP` seguito da una ricreazione, ma senza ricreare l'indice.

------------------------------------------------------------------------

## ⚡ La soluzione

Dieci minuti. Dal momento in cui mi ero collegato a quando ho identificato la causa. Non per bravura — per gli strumenti.

La fix era semplice:

``` sql
CREATE INDEX idx_movimenti_temp_cliente
ON movimenti_temp (id_cliente, data_movimento)
TABLESPACE idx_data;
```

Dopo la creazione dell'indice, ho forzato un re-parse della query:

``` sql
EXEC DBMS_SHARED_POOL.purge('g4f2h8k1nw3z9', 'C');
```

Ho chiesto al team di rilanciare il batch. Tempo di esecuzione: 18 minuti. Identico ai test precedenti.

Il go-live del sabato mattina è andato regolarmente.

------------------------------------------------------------------------

## 📋 AWR vs ASH: quando usare cosa

Dopo quell'episodio ho formalizzato una regola che uso sempre:

| Caratteristica | AWR | ASH |
|---|---|---|
| Granularità | Snapshot orari (configurabili) | Campione ogni secondo |
| Profondità storica | Fino a 30 giorni (default 8) | 1 ora in memoria, poi in AWR |
| Caso d'uso principale | Trend analysis, confronto periodi | Diagnosi puntuale, isolamento SQL |
| Vista principale | `DBA_HIST_*` | `V$ACTIVE_SESSION_HISTORY` |
| Vista storica | — | `DBA_HIST_ACTIVE_SESS_HISTORY` |
| Licenza richiesta | Diagnostic Pack | Diagnostic Pack |
| Output tipico | Report HTML/text | Query ad hoc |

La regola empirica: **AWR per capire cosa è cambiato, ASH per capire perché**.

AWR ti dice: "Tra le 17:00 e le 18:00, il 58% del tempo del database è stato speso in full table scan." ASH ti dice: "Alle 17:12:34, la sessione 847 stava eseguendo la query g4f2h8k1nw3z9 con un full table scan su MOVIMENTI_TEMP, lanciata dal programma BatchVerificaProduzione."

Sono complementari. Usarne solo uno è come diagnosticare un problema guardando solo la TAC o solo gli esami del sangue.

------------------------------------------------------------------------

## 🛡️ Le query che ogni DBA dovrebbe avere pronte

Negli anni ho costruito un set di query diagnostiche che tengo sempre a portata di mano. Le condivido perché in un'emergenza non c'è tempo per scriverle da zero.

### Top SQL per tempo di esecuzione (ultima ora)

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

### Wait event distribution per un SQL specifico

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

### Confronto piani di esecuzione nel tempo

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

## 🎯 Cosa ho imparato quella sera

Tre lezioni che porto con me.

**Prima**: il deploy non è solo codice. È anche struttura. Quando rilasci in produzione, devi verificare che indici, constraint, statistiche e grant siano coerenti con quello che c'era prima. Uno script che fa `DROP TABLE` e `CREATE TABLE` senza ricreare gli indici è una bomba a orologeria.

**Seconda**: AWR e ASH non sono strumenti per DBA senior. Sono strumenti di prima linea, come un defibrillatore. Devi saperli usare prima di averne bisogno, non durante l'emergenza.

**Terza**: dieci minuti di diagnosi corretta valgono più di tre ore di tentativi alla cieca. Quando il sistema è in ginocchio, la tentazione è riavviare, uccidere sessioni, aumentare risorse. Ma senza sapere cosa sta succedendo, stai sparando nel buio.

Quella sera sono uscito dall'ufficio alle 19:20. Quaranta minuti dopo la telefonata. Il giorno dopo il go-live è partito senza intoppi, e il lunedì il sistema girava regolarmente.

Non sono un eroe. Ho solo usato gli strumenti giusti.

------------------------------------------------------------------------

## Glossario

**[AWR](/it/glossary/awr/)** — Automatic Workload Repository. Componente integrato in Oracle che raccoglie statistiche di performance tramite snapshot periodici e genera report diagnostici comparativi.

**[ASH](/it/glossary/ash/)** — Active Session History. Componente Oracle che campiona lo stato di ogni sessione attiva una volta al secondo, conservandolo in memoria e poi in AWR. È il microscopio del DBA per la diagnosi puntuale.

**[Full Table Scan](/it/glossary/full-table-scan/)** — Operazione di lettura in cui Oracle legge tutti i blocchi di una tabella senza usare indici. Nei wait event compare come `db file scattered read`.

**[Wait Event](/it/glossary/wait-event/)** — Evento di attesa registrato da Oracle ogni volta che una sessione non può procedere perché attende una risorsa (I/O, lock, CPU, rete). L'analisi dei wait event è la base della metodologia diagnostica Oracle.

**[Snapshot](/it/glossary/snapshot-oracle/)** — Istantanea delle statistiche di performance catturata periodicamente da AWR (di default ogni 60 minuti). Il confronto tra due snapshot genera il report AWR.
