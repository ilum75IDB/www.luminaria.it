---
title: "Oracle Partitioning: când 2 miliarde de rânduri nu mai încap într-o interogare"
description: "Un client cu o tabelă de tranzacții de 2 miliarde de rânduri și interogări de raportare care trecuseră de la secunde la ore. Cum am rezolvat cu partitioning Oracle — range, interval, partition pruning și indecși locali."
date: "2025-12-30T10:00:00+01:00"
draft: false
translationKey: "oracle_partitioning"
tags: ["oracle", "partitioning", "performance", "tuning", "execution-plan", "range-partitioning"]
categories: ["oracle", "performance"]
image: "oracle-partitioning.cover.jpg"
---

Două miliarde de rânduri. Nu este un număr la care ajungi într-o zi. Durează ani de tranzacții, mișcări, înregistrări zilnice care se acumulează. Și în tot acest timp baza de date funcționează, interogările răspund, rapoartele ies. Apoi într-o zi cineva deschide un ticket: „raportul lunar durează patru ore."

Patru ore. Pentru un raport care cu șase luni înainte dura douăzeci de minute.

Nu este un bug. Nu este o problemă de rețea sau de stocare lentă. Este fizica datelor: când o tabelă crește peste un anumit prag, abordările care funcționau nu mai funcționează. Și dacă nu ai proiectat structura să gestioneze acea creștere, baza de date face singurul lucru pe care îl poate face: citește totul.

---

## Contextul: telecomunicații și volume industriale

Clientul era un operator de telecomunicații. Nimic exotic — un clasic mediu Oracle 19c Enterprise Edition pe Linux, stocare SAN, vreo treizeci de instanțe între producție, staging și dezvoltare. Instanța critică era cea de facturare: facturare, CDR (Call Detail Records), mișcări contabile.

Tabela din centrul problemei se numea `TXN_MOVIMENTI`. Colecta fiecare tranzacție individuală din sistemul de facturare din 2016. Structura era aproximativ aceasta:

``` sql
CREATE TABLE txn_movimenti (
    txn_id         NUMBER(18)     NOT NULL,
    data_movimento DATE           NOT NULL,
    cod_cliente    VARCHAR2(20)   NOT NULL,
    tipo_movimento VARCHAR2(10)   NOT NULL,
    importo        NUMBER(15,4),
    canale         VARCHAR2(30),
    stato          VARCHAR2(5)    DEFAULT 'ATT',
    data_insert    TIMESTAMP      DEFAULT SYSTIMESTAMP,
    CONSTRAINT pk_txn_movimenti PRIMARY KEY (txn_id)
);
```

2,1 miliarde de rânduri. 380 GB de date. Un singur segment, un singur tablespace, fără partiții. Un monolit.

Indecșii existau: unul pe cheia primară, unul pe `data_movimento`, unul compozit pe `(cod_cliente, data_movimento)`. Dar când o tabelă depășește o anumită dimensiune, chiar și un index range scan nu mai este suficient, pentru că volumul de date returnat este în continuare enorm.

---

## Simptomele: nu este lentoare, este colaps

Problemele nu au apărut toate odată. Au venit treptat, cum se întâmplă întotdeauna cu tabelele care cresc fără control.

**Primul semnal**: rapoartele lunare. Interogarea agregată de facturare — care suma importurile per client pentru o lună dată — trecuse de la 20 de minute la 4 ore în decursul unui an. Planul de execuție arăta un index range scan pe dată, dar numărul de blocuri citite era monstruos: Oracle trebuia să parcurgă sute de mii de leaf block-uri ale indexului și apoi să facă table access by rowid pentru a recupera coloanele neacoperite.

**Al doilea semnal**: mentenanța. `ALTER INDEX REBUILD` pe indexul datei dura șase ore. Colectarea statisticilor (`DBMS_STATS.GATHER_TABLE_STATS`) nu se termina într-o noapte. Backup-urile RMAN deveniseră o ruletă: uneori intrau în fereastră, alteori nu.

**Al treilea semnal**: full table scan-urile involuntare. Interogări cu predicate pe dată pe care optimizatorul decidea să le rezolve cu un full table scan pentru că costul estimat al index scan-ului era superior. Pe 380 GB de date.

Planul de execuție al interogării de facturare era acesta:

``` sql
SELECT cod_cliente,
       TRUNC(data_movimento, 'MM') AS mese,
       SUM(importo) AS totale
FROM   txn_movimenti
WHERE  data_movimento BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'
AND    stato = 'CON'
GROUP BY cod_cliente, TRUNC(data_movimento, 'MM');
```

``` text
---------------------------------------------------------------------
| Id  | Operation                    | Name            | Rows  | Cost |
---------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                 |  125K | 890K |
|   1 |  HASH GROUP BY               |                 |  125K | 890K |
|   2 |   TABLE ACCESS BY INDEX ROWID| TXN_MOVIMENTI   |  28M  | 885K |
|*  3 |    INDEX RANGE SCAN          | IDX_TXN_DATA    |  28M  | 85K  |
---------------------------------------------------------------------
```

28 de milioane de rânduri doar pentru ianuarie. Indexul găsea rândurile, dar apoi Oracle trebuia să extragă fiecare rând individual din tabelă pentru a citi `cod_cliente`, `importo` și `stato`. Milioane de operații de I/O aleatoriu pe o tabelă de 380 GB dispersată pe mii de blocuri.

---

## Soluția: nu ai nevoie de un index mai bun, ai nevoie de o structură diferită

Am petrecut două zile analizând pattern-urile de acces înainte de a propune orice soluție. Pentru că partitioning-ul nu este o baghetă magică — dacă alegi greșit cheia de partiție, înrăutățești lucrurile.

Pattern-urile erau clare:

- **90% din interogări** aveau un predicat pe dată (`data_movimento`)
- Rapoartele erau întotdeauna **lunare sau trimestriale**
- Interogările operaționale (client individual) foloseau întotdeauna `cod_cliente + data_movimento`
- Datele mai vechi de 3 ani nu erau niciodată citite de rapoarte, doar de procesele anuale de arhivare

Alegerea a căzut pe un **interval partitioning lunar** pe coloana `data_movimento`. Nu range partitioning clasic, unde trebuie să creezi manual fiecare partiție viitoare. Interval: definești intervalul o singură dată și Oracle creează partițiile automat când sosesc date pentru o nouă perioadă.

---

## Implementarea: CTAS, indecși locali și zero downtime (aproape)

Nu poți face `ALTER TABLE ... PARTITION BY` pe o tabelă existentă cu 2 miliarde de rânduri. Nu în Oracle 19c, cel puțin nu fără Online Table Redefinition. Și acea opțiune, pe o tabelă de aceste dimensiuni, are propriile riscuri.

Am ales abordarea CTAS — Create Table As Select — cu paralelism. Creezi noua tabelă partițională, copiezi datele, redenumești.

### Pasul 1: crearea tabelei partiționate

``` sql
CREATE TABLE txn_movimenti_part
PARTITION BY RANGE (data_movimento)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    PARTITION p_before_2016 VALUES LESS THAN (DATE '2016-01-01'),
    PARTITION p_2016_01     VALUES LESS THAN (DATE '2016-02-01'),
    PARTITION p_2016_02     VALUES LESS THAN (DATE '2016-03-01')
    -- Oracle va crea automat partițiile ulterioare
)
TABLESPACE ts_billing_data
NOLOGGING
PARALLEL 8
AS
SELECT /*+ PARALLEL(t, 8) */
       txn_id, data_movimento, cod_cliente, tipo_movimento,
       importo, canale, stato, data_insert
FROM   txn_movimenti t;
```

`NOLOGGING` este fundamental: fără el, copia generează redo log pentru fiecare bloc scris. Pe 380 GB ar însemna umplerea zonei de redo și punerea sistemului în mod archivelog timp de zile. Cu `NOLOGGING` copia a durat 3 ore și jumătate cu paralelism la 8.

După copiere am restaurat logging-ul:

``` sql
ALTER TABLE txn_movimenti_part LOGGING;
```

Și am lansat un backup RMAN imediat, pentru că segmentele NOLOGGING nu sunt recuperabile în caz de restore.

### Pasul 2: indecși locali

Designul indecșilor pe o tabelă partițională este diferit de o tabelă normală. Conceptul cheie este: **index local vs index global**.

Un index **local** este partiționat cu aceeași cheie ca tabela. Fiecare partiție a tabelei are partiția de index corespunzătoare. Avantaj: operațiile de mentenanță pe o partiție nu le afectează pe celelalte.

Un index **global** acoperă toate partițiile. Este mai eficient pentru interogări care nu filtrează pe cheia de partiție, dar orice operație DDL pe partiție (drop, truncate, split) invalidează indexul întreg.

``` sql
-- Cheie primară ca index global (necesar pentru lookup-uri punctuale)
ALTER TABLE txn_movimenti_part
ADD CONSTRAINT pk_txn_mov_part PRIMARY KEY (txn_id)
USING INDEX GLOBAL;

-- Index local pe dată (aliniat cu partiția)
CREATE INDEX idx_txn_mov_data ON txn_movimenti_part (data_movimento)
LOCAL PARALLEL 8;

-- Index local compozit pentru interogări operaționale
CREATE INDEX idx_txn_mov_cli_data
ON txn_movimenti_part (cod_cliente, data_movimento)
LOCAL PARALLEL 8;
```

Cheia primară rămâne globală pentru că interogările pe `txn_id` nu includ niciodată data — ai nevoie de acces direct. Ceilalți indecși sunt locali pentru că se aliniază cu pattern-urile de utilizare: interogări pe dată, interogări pe client+dată.

### Pasul 3: comutarea

``` sql
-- Redenumirea tabelei originale (backup)
ALTER TABLE txn_movimenti RENAME TO txn_movimenti_old;

-- Redenumirea noii tabele
ALTER TABLE txn_movimenti_part RENAME TO txn_movimenti;

-- Reconstruirea sinonimelor dacă există
-- Recompilarea obiectelor invalidate
BEGIN
  FOR obj IN (SELECT object_name, object_type
              FROM   dba_objects
              WHERE  status = 'INVALID'
              AND    owner = 'BILLING') LOOP
    BEGIN
      IF obj.object_type = 'PACKAGE BODY' THEN
        EXECUTE IMMEDIATE 'ALTER PACKAGE billing.'
          || obj.object_name || ' COMPILE BODY';
      ELSIF obj.object_type IN ('PROCEDURE','FUNCTION','VIEW') THEN
        EXECUTE IMMEDIATE 'ALTER ' || obj.object_type
          || ' billing.' || obj.object_name || ' COMPILE';
      END IF;
    EXCEPTION WHEN OTHERS THEN NULL;
    END;
  END LOOP;
END;
/
```

Downtime-ul efectiv a fost timpul celor două `ALTER TABLE RENAME`: câteva secunde. Tot restul — copia datelor, crearea indecșilor — s-a întâmplat în paralel cu sistemul activ.

### Pasul 4: colectarea statisticilor

``` sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname          => 'BILLING',
    tabname          => 'TXN_MOVIMENTI',
    granularity      => 'ALL',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
    degree           => 8
  );
END;
/
```

Parametrul `granularity => 'ALL'` este important: îi spune lui Oracle să colecteze statistici la nivel global, de partiție și de subpartiție. Fără el, optimizatorul ar putea lua decizii greșite pentru că nu cunoaște distribuția datelor în interiorul partițiilor individuale.

---

## Înainte și după: cifrele

Aceeași interogare de facturare, după partitioning:

``` text
------------------------------------------------------------------------
| Id  | Operation                       | Name            | Rows  | Cost |
------------------------------------------------------------------------
|   0 | SELECT STATEMENT                |                 |  125K | 12K  |
|   1 |  HASH GROUP BY                  |                 |  125K | 12K  |
|   2 |   PARTITION RANGE SINGLE        |                 |  28M  | 11K  |
|   3 |    TABLE ACCESS FULL            | TXN_MOVIMENTI   |  28M  | 11K  |
------------------------------------------------------------------------
```

Uitați-vă la operația de la pasul 2: `PARTITION RANGE SINGLE`. Oracle știe că datele din ianuarie stau într-o singură partiție și citește doar aceea. Full table scan-ul care înainte teroriza este acum un full **partition** scan — pe aproximativ 4 GB în loc de 380.

| Metrică | Înainte | După | Variație |
|---|---|---|---|
| Timp interogare lunară | 4 ore | 3 minute | -98% |
| Consistent gets | 48M | 580K | -98.8% |
| Physical reads | 12M | 95K | -99.2% |
| Timp GATHER_TABLE_STATS | 14 ore | 25 min (per partiție) | -97% |
| Timp rebuild index | 6 ore | 12 min (per partiție) | -97% |
| Dimensiune backup incremental | 380 GB | ~4 GB/lună | -99% |

Costul a trecut de la 890K la 12K. Nu este o îmbunătățire procentuală — este un ordin de mărime diferit.

---

## Partition pruning: adevărata magie

Mecanismul care face toate acestea posibile se numește **partition pruning**. Nu este ceva ce trebuie configurat — Oracle o face automat când predicatul interogării corespunde cheii de partiție.

Dar trebuie să știi când funcționează și când nu.

**Funcționează** cu predicate directe pe coloana de partiție:

``` sql
-- Pruning activ: Oracle citește doar partiția din ianuarie
WHERE data_movimento BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'

-- Pruning activ: Oracle citește doar partiția specifică
WHERE data_movimento = DATE '2025-03-15'
```

**Nu funcționează** când coloana este în interiorul unei funcții:

``` sql
-- Pruning DEZACTIVAT: Oracle trebuie să citească toate partițiile
WHERE TRUNC(data_movimento) = DATE '2025-01-01'

-- Pruning DEZACTIVAT: funcție pe coloană
WHERE TO_CHAR(data_movimento, 'YYYY-MM') = '2025-01'

-- Pruning DEZACTIVAT: expresie aritmetică
WHERE data_movimento + 30 > SYSDATE
```

Aceasta este eroarea cea mai comună pe care o văd după o implementare de partitioning: dezvoltatorii aplică funcții pe coloana de dată fără să realizeze că dezactivează pruning-ul. Și tabela revine la a fi citită integral.

Am dedicat o jumătate de zi revizuirii tuturor interogărilor aplicației care atingeau `TXN_MOVIMENTI`. Am găsit unsprezece cu `TRUNC(data_movimento)` în `WHERE`. Unsprezece interogări care ar fi ignorat partitioning-ul.

---

## Gestionarea ciclului de viață: drop partition

Unul dintre avantajele cele mai concrete ale partitioning-ului este gestionarea ciclului de viață al datelor. Înainte de partitioning, arhivarea datelor vechi însemna un `DELETE` de miliarde de rânduri — o operație care generează munți de redo și undo, blochează tabela ore întregi și riscă să facă să explodeze tablespace-ul de undo.

Cu partitioning:

``` sql
-- Arhivarea datelor din 2016 într-un tablespace read-only
ALTER TABLE txn_movimenti
MOVE PARTITION p_2016_01 TABLESPACE ts_archive;

-- Sau, dacă datele nu mai sunt necesare
ALTER TABLE txn_movimenti DROP PARTITION p_2016_01;
```

Un `DROP PARTITION` pe o partiție de 4 GB durează mai puțin de o secundă. Nu generează undo. Nu generează redo semnificativ. Nu blochează celelalte partiții. Este o operație DDL, nu DML.

Am configurat un job lunar care muta partițiile mai vechi de 5 ani în tablespace-ul de arhivă și le punea în read-only. Clientul a recuperat 120 GB de spațiu activ fără să șteargă un singur dat.

---

## Ce am învățat (și erorile de evitat)

După cincisprezece ani de partitioning Oracle, am o listă de lucruri pe care mi-aș fi dorit să le fi știut mai devreme.

**Cheia de partiție trebuie să corespundă pattern-ului de acces.** Pare evident, dar am văzut tabele partiționate pe `cod_cliente` când 95% din interogări filtrează pe dată. Partitioning-ul funcționează doar dacă interogările pot face pruning.

**Interval partitioning este aproape întotdeauna mai bun decât range static.** Cu range clasic trebuie să creezi manual partițiile viitoare, ceea ce înseamnă un job programat sau un DBA care și-l amintește. Cu interval Oracle le creează singur. O problemă mai puțin.

**Indecșii globali sunt o capcană.** Funcționează bine pentru interogări, dar orice operație DDL pe partiție îi invalidează. Și reconstruirea unui index global pe 2 miliarde de rânduri durează ore. Folosește indecși locali unde este posibil și acceptă compromisul.

**NOLOGGING nu este opțional pentru operații masive.** Fără NOLOGGING, un CTAS de 380 GB generează aceeași cantitate de redo. Zona ta de archivelog se va umple, baza de date va intra în așteptare, iar DBA-ul de gardă va primi un telefon la 3 dimineața.

**Testează pruning-ul înainte de a merge în producție.** Nu te încrede: verifică cu `EXPLAIN PLAN` că fiecare interogare critică face efectiv pruning. Un singur `TRUNC()` în predicatul greșit și ai un full table scan de 380 GB.

**Partitioning-ul nu înlocuiește indecșii.** Reduce volumul de date de examinat, dar în interiorul partiției ai nevoie în continuare de indecșii potriviți. O partiție lunară de 28 de milioane de rânduri fără index este tot o problemă.

---

## Când ai nevoie de partitioning

Nu toate tabelele au nevoie de partitioning. Regula mea empirică:

- Sub 10 milioane de rânduri: probabil nu
- Între 10 și 100 de milioane: depinde de pattern-ul de acces și de ritmul de creștere
- Peste 100 de milioane: probabil da
- Peste un miliard: nu ai de ales

Dar momentul potrivit pentru a-l implementa este înainte să devină urgent. Când tabela are deja 2 miliarde de rânduri, migrarea este un proiect în sine. Când are 50 de milioane și crește, este treabă de o după-amiază.

Cea mai mare eroare a mea cu partitioning-ul? Că nu l-am propus cu șase luni mai devreme, când toate semnalele erau deja acolo.
