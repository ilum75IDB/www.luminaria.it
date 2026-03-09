---
title: "Oracle Partitioning: quando 2 miliardi di righe non entrano più in una query"
description: "Un cliente con una tabella transazioni da 2 miliardi di righe e query di reportistica che da secondi erano passate a ore. Come ho risolto con il partitioning Oracle — range, interval, partition pruning e indici locali."
date: "2025-12-23T10:00:00+01:00"
draft: false
translationKey: "oracle_partitioning"
tags: ["partitioning", "performance", "tuning", "execution-plan"]
categories: ["oracle"]
image: "oracle-partitioning.cover.jpg"
---

Due miliardi di righe. Non è un numero che si raggiunge in un giorno. Ci vogliono anni di transazioni, di movimenti, di registrazioni quotidiane che si accumulano. E per tutto quel tempo il database funziona, le query rispondono, i report escono. Poi un giorno qualcuno apre un ticket: "il report mensile ci mette quattro ore."

Quattro ore. Per un report che sei mesi prima ne impiegava venti minuti.

Non è un bug. Non è un problema di rete o di storage lento. È la fisica dei dati: quando una tabella cresce oltre una certa soglia, gli approcci che funzionavano smettono di funzionare. E se non hai progettato la struttura per gestire quella crescita, il database fa l'unica cosa che può fare: leggere tutto.

---

## Il contesto: telecomunicazioni e volumi industriali

Il cliente era un operatore telecom. Niente di esotico — un classico ambiente Oracle 19c Enterprise Edition su Linux, storage SAN, una trentina di istanze tra produzione, staging e sviluppo. L'istanza critica era quella del billing: fatturazione, CDR (Call Detail Records), movimenti contabili.

La tabella al centro del problema si chiamava `TXN_MOVIMENTI`. Raccoglieva ogni singola transazione del sistema di billing dal 2016. La struttura era più o meno questa:

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

2.1 miliardi di righe. 380 GB di dati. Un solo segmento, un solo tablespace, nessuna partizione. Un monolite.

Gli indici c'erano: uno sulla primary key, uno su `data_movimento`, uno composito su `(cod_cliente, data_movimento)`. Ma quando la tabella supera una certa dimensione, anche un index range scan non basta più, perché il volume di dati restituito è comunque enorme.

---

## I sintomi: non è lentezza, è collasso

I problemi non si sono presentati tutti insieme. Sono arrivati gradualmente, come succede sempre con le tabelle che crescono senza controllo.

**Primo segnale**: i report mensili. La query di fatturazione aggregata — che sommava gli importi per cliente per un dato mese — era passata da 20 minuti a 4 ore nell'arco di un anno. L'execution plan mostrava un index range scan sulla data, ma il numero di blocchi letti era mostruoso: Oracle doveva attraversare centinaia di migliaia di leaf block dell'indice e poi fare le table access by rowid per recuperare le colonne non coperte.

**Secondo segnale**: la manutenzione. L'`ALTER INDEX REBUILD` sull'indice della data richiedeva sei ore. Le statistiche (`DBMS_STATS.GATHER_TABLE_STATS`) non terminavano in una notte. I backup RMAN erano diventati una roulette: a volte entravano nella finestra, a volte no.

**Terzo segnale**: i full table scan involontari. Query con predicati sulla data che l'optimizer decideva di risolvere con un full table scan perché il costo stimato dell'index scan era superiore. Su 380 GB di dati.

L'execution plan della query di fatturazione era questo:

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

28 milioni di righe solo per gennaio. L'indice trovava le righe, ma poi Oracle doveva andare a pescare ogni singola riga dalla tabella per leggere `cod_cliente`, `importo` e `stato`. Milioni di random I/O su una tabella da 380 GB sparsa su migliaia di blocchi.

---

## La soluzione: non serve un indice migliore, serve una struttura diversa

Ho passato due giorni ad analizzare i pattern di accesso prima di proporre qualsiasi soluzione. Perché il partitioning non è una bacchetta magica — se sbagli la chiave di partizione, peggiori le cose.

I pattern erano chiari:

- Il **90% delle query** aveva un predicato sulla data (`data_movimento`)
- I report erano sempre **mensili o trimestrali**
- Le query operative (singolo cliente) usavano sempre `cod_cliente + data_movimento`
- I dati oltre i 3 anni non venivano mai letti dai report, solo dai batch annuali di archiviazione

La scelta è caduta su un **interval partitioning mensile** sulla colonna `data_movimento`. Non range partitioning classico, dove devi creare manualmente ogni partizione futura. Interval: definisci l'intervallo una volta e Oracle crea le partizioni automaticamente quando arrivano dati per un nuovo periodo.

---

## L'implementazione: CTAS, indici locali e zero downtime (quasi)

Non puoi fare `ALTER TABLE ... PARTITION BY` su una tabella esistente con 2 miliardi di righe. Non in Oracle 19c, almeno non senza l'opzione Online Table Redefinition. E quella opzione, su una tabella di queste dimensioni, ha i suoi rischi.

Ho scelto l'approccio CTAS — Create Table As Select — con parallelismo. Creare la nuova tabella partizionata, copiarci i dati, rinominare.

### Step 1: creare la tabella partizionata

``` sql
CREATE TABLE txn_movimenti_part
PARTITION BY RANGE (data_movimento)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    PARTITION p_before_2016 VALUES LESS THAN (DATE '2016-01-01'),
    PARTITION p_2016_01     VALUES LESS THAN (DATE '2016-02-01'),
    PARTITION p_2016_02     VALUES LESS THAN (DATE '2016-03-01')
    -- Oracle creerà automaticamente le partizioni successive
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

Il `NOLOGGING` è fondamentale: senza di esso la copia genera redo log per ogni blocco scritto. Su 380 GB significherebbe riempire l'area di redo e mandare in archivelog il sistema per giorni. Con `NOLOGGING` la copia è andata in 3 ore e mezza con parallelismo a 8.

Ovviamente dopo la copia ho rimesso il logging:

``` sql
ALTER TABLE txn_movimenti_part LOGGING;
```

E ho lanciato un backup RMAN immediatamente, perché i segmenti NOLOGGING non sono recuperabili in caso di restore.

### Step 2: indici locali

La scelta degli indici su una tabella partizionata è diversa da una tabella normale. Il concetto chiave è: **indice locale vs indice globale**.

Un indice **locale** è partizionato con la stessa chiave della tabella. Ogni partizione della tabella ha la sua partizione di indice corrispondente. Vantaggio: le operazioni di manutenzione su una partizione non toccano le altre.

Un indice **globale** copre tutte le partizioni. È più efficiente per query che non filtrano sulla chiave di partizione, ma qualsiasi operazione DDL sulla partizione (drop, truncate, split) invalida l'indice intero.

``` sql
-- Primary key come indice globale (serve per lookup puntuali)
ALTER TABLE txn_movimenti_part
ADD CONSTRAINT pk_txn_mov_part PRIMARY KEY (txn_id)
USING INDEX GLOBAL;

-- Indice locale sulla data (partition-aligned)
CREATE INDEX idx_txn_mov_data ON txn_movimenti_part (data_movimento)
LOCAL PARALLEL 8;

-- Indice locale composito per le query operative
CREATE INDEX idx_txn_mov_cli_data
ON txn_movimenti_part (cod_cliente, data_movimento)
LOCAL PARALLEL 8;
```

La primary key resta globale perché le query per `txn_id` non includono mai la data — serve un accesso diretto. Gli altri indici sono locali perché allineati con il pattern d'uso: query per data, query per cliente+data.

### Step 3: lo switch

``` sql
-- Rinominare la tabella originale (backup)
ALTER TABLE txn_movimenti RENAME TO txn_movimenti_old;

-- Rinominare la nuova tabella
ALTER TABLE txn_movimenti_part RENAME TO txn_movimenti;

-- Ricostruire i sinonimi se presenti
-- Ricompilare gli oggetti invalidati
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

Il downtime effettivo è stato il tempo dei due `ALTER TABLE RENAME`: qualche secondo. Tutto il resto — la copia dei dati, la creazione degli indici — è avvenuto in parallelo con il sistema attivo.

### Step 4: raccogliere le statistiche

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

Il parametro `granularity => 'ALL'` è importante: dice a Oracle di raccogliere statistiche a livello globale, di partizione e di subpartizione. Senza questo, l'optimizer potrebbe prendere decisioni sbagliate perché non conosce la distribuzione dei dati dentro le singole partizioni.

---

## Prima e dopo: i numeri

La stessa query di fatturazione, dopo il partitioning:

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

Guardate l'operazione al passo 2: `PARTITION RANGE SINGLE`. Oracle sa che i dati di gennaio stanno in una sola partizione e legge solo quella. Il full table scan che prima terrorizzava è ora un full **partition** scan — su circa 4 GB invece di 380.

| Metrica | Prima | Dopo | Variazione |
|---|---|---|---|
| Tempo query mensile | 4 ore | 3 minuti | -98% |
| Consistent gets | 48M | 580K | -98.8% |
| Physical reads | 12M | 95K | -99.2% |
| Tempo GATHER_TABLE_STATS | 14 ore | 25 min (per partizione) | -97% |
| Tempo rebuild indice | 6 ore | 12 min (per partizione) | -97% |
| Spazio backup incrementale | 380 GB | ~4 GB/mese | -99% |

Il costo è passato da 890K a 12K. Non è un miglioramento percentuale — è un ordine di grandezza diverso.

---

## Partition pruning: la vera magia

Il meccanismo che rende tutto questo possibile si chiama **partition pruning**. Non è qualcosa che devi configurare — Oracle lo fa automaticamente quando il predicato della query corrisponde alla chiave di partizione.

Ma devi sapere quando funziona e quando no.

**Funziona** con predicati diretti sulla colonna di partizione:

``` sql
-- Pruning attivo: Oracle legge solo la partizione di gennaio
WHERE data_movimento BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'

-- Pruning attivo: Oracle legge solo la partizione specifica
WHERE data_movimento = DATE '2025-03-15'
```

**Non funziona** quando la colonna è dentro una funzione:

``` sql
-- Pruning DISATTIVATO: Oracle deve leggere tutte le partizioni
WHERE TRUNC(data_movimento) = DATE '2025-01-01'

-- Pruning DISATTIVATO: funzione sulla colonna
WHERE TO_CHAR(data_movimento, 'YYYY-MM') = '2025-01'

-- Pruning DISATTIVATO: espressione aritmetica
WHERE data_movimento + 30 > SYSDATE
```

Questo è l'errore più comune che vedo dopo un'implementazione di partitioning: gli sviluppatori applicano funzioni alla colonna di data senza rendersi conto che stanno disabilitando il pruning. E la tabella torna a essere letta per intero.

Ho dedicato mezza giornata a fare una review di tutte le query dell'applicazione che toccavano `TXN_MOVIMENTI`. Ne ho trovate undici con `TRUNC(data_movimento)` nel `WHERE`. Undici query che avrebbero ignorato il partitioning.

---

## La gestione del ciclo di vita: drop partition

Uno dei vantaggi più concreti del partitioning è la gestione del ciclo di vita dei dati. Prima del partitioning, archiviare i dati vecchi significava un `DELETE` da miliardi di righe — operazione che generava montagne di redo e undo, bloccava la tabella per ore e rischiava di far esplodere il tablespace di undo.

Con il partitioning:

``` sql
-- Archiviare i dati del 2016 su un tablespace read-only
ALTER TABLE txn_movimenti
MOVE PARTITION p_2016_01 TABLESPACE ts_archive;

-- Oppure, se i dati non servono più
ALTER TABLE txn_movimenti DROP PARTITION p_2016_01;
```

Un `DROP PARTITION` su una partizione da 4 GB richiede meno di un secondo. Non genera undo. Non genera redo significativo. Non blocca le altre partizioni. È un'operazione DDL, non DML.

Ho impostato un job mensile che spostava le partizioni più vecchie di 5 anni nel tablespace di archivio e le metteva in read-only. Il cliente ha recuperato 120 GB di spazio attivo senza cancellare un singolo dato.

---

## Cosa ho imparato (e gli errori da evitare)

Dopo quindici anni di partitioning Oracle, ho una lista di cose che vorrei aver saputo prima.

**La chiave di partizione deve corrispondere al pattern di accesso.** Sembra ovvio, ma ho visto tabelle partizionate per `cod_cliente` quando il 95% delle query filtra per data. Il partitioning funziona solo se le query possono fare pruning.

**Interval partitioning è quasi sempre meglio di range statico.** Con il range classico devi creare manualmente le partizioni future, il che significa un job schedulato o un DBA che se lo ricorda. Con interval Oracle le crea da solo. Un problema in meno.

**Gli indici globali sono una trappola.** Funzionano bene per le query, ma qualsiasi operazione DDL sulla partizione li invalida. E ricostruire un indice globale su 2 miliardi di righe richiede ore. Usa indici locali dove possibile e accetta il compromesso.

**NOLOGGING non è opzionale per le operazioni bulk.** Senza NOLOGGING, un CTAS di 380 GB genera altrettanti redo. Il tuo archivelog si riempirà, il database andrà in wait, e il DBA di turno riceverà una chiamata alle 3 di notte.

**Testa il pruning prima di andare in produzione.** Non fidarti: verifica con `EXPLAIN PLAN` che ogni query critica faccia effettivamente pruning. Una singola `TRUNC()` nel predicato sbagliato e hai 380 GB di full table scan.

**Il partitioning non sostituisce gli indici.** Riduce il volume di dati da esaminare, ma dentro la partizione hai ancora bisogno degli indici giusti. Una partizione mensile da 28 milioni di righe senza indice è comunque un problema.

---

## Quando hai bisogno del partitioning

Non tutte le tabelle hanno bisogno di partitioning. La mia regola empirica:

- Sotto i 10 milioni di righe: probabilmente no
- Tra 10 e 100 milioni: dipende dal pattern di accesso e dalla crescita
- Sopra i 100 milioni: probabilmente sì
- Sopra il miliardo: non hai scelta

Ma il momento giusto per implementarlo è prima che diventi urgente. Quando la tabella ha già 2 miliardi di righe, la migrazione è un progetto a sé. Quando ne ha 50 milioni e sta crescendo, è un'operazione da un pomeriggio.

Il mio errore più grande con il partitioning? Non averlo proposto sei mesi prima, quando i segnali c'erano già tutti.
