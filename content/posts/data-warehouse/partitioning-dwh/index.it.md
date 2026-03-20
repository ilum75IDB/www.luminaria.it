---
title: "Partitioning nel DWH: quando 3 anni di dati pesano troppo"
description: "Una fact table da 800 milioni di righe senza partitioning, query trimestrali che giravano per 12 minuti e un business che voleva risposte in tempo reale. Come ho implementato il range partitioning per mese e portato i tempi a 40 secondi."
date: "2026-04-07T10:00:00+01:00"
draft: false
translationKey: "partitioning_dwh"
tags: ["partitioning", "performance", "oracle", "fact-table", "data-warehouse"]
categories: ["data-warehouse"]
image: "partitioning-dwh.cover.jpg"
---

La settimana scorsa un collega mi ha raccontato di un progetto dove le query sul data warehouse avevano smesso di tornare in tempi ragionevoli. "Quanto ci mette il report trimestrale?" gli ho chiesto. "Dodici minuti." "E prima?" "Un minuto e mezzo."

Non ho dovuto chiedere altro. Conoscevo già il copione.

Una {{< glossary term="fact-table" >}}fact table{{< /glossary >}} che parte piccola, cresce ogni giorno, e nessuno si preoccupa della struttura fisica finché un giorno le query non tornano più. Non è un bug, non è un errore di codice. È il peso dei dati che alla fine si fa sentire.

---

## Il contesto: GDO e tre anni di scontrini

Il progetto era nel settore della grande distribuzione organizzata — una catena di supermercati con circa duecento punti vendita, un centinaio di milioni di euro di fatturato annuo, e un data warehouse Oracle 19c che raccoglieva tutto: vendite, resi, movimenti di magazzino, promozioni.

La tabella al centro del problema si chiamava `FACT_VENDITE`. Ogni riga era una riga di scontrino — uno scontrino medio ha otto righe, moltiplicato per trentamila scontrini al giorno su duecento negozi, fa circa 48 milioni di righe al mese. In tre anni si erano accumulate 800 milioni di righe.

La struttura era questa:

```sql
CREATE TABLE fact_vendite (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20),
    CONSTRAINT pk_fact_vendite PRIMARY KEY (vendita_id)
);
```

Un unico indice sulla primary key, un indice su `data_vendita` e uno composito su `(punto_vendita_id, data_vendita)`. Nessun partizionamento. Otto centinaia di milioni di righe in un'unica tabella monolitica.

## 🔍 Il sintomo: full table scan su 800 milioni di righe

Le query analitiche del DWH lavoravano quasi sempre per periodo. Vendite dell'ultimo trimestre per punto vendita. Confronto anno su anno per categoria merceologica. Margini mensili per regione. Tutte query con un filtro su `data_vendita`.

Il report trimestrale era questo:

```sql
SELECT pv.regione,
       cat.famiglia,
       SUM(f.importo)    AS fatturato,
       SUM(f.quantita)   AS pezzi_venduti,
       SUM(f.sconto)     AS sconto_totale
FROM   fact_vendite f
JOIN   dim_punto_vendita pv  ON f.punto_vendita_id = pv.punto_vendita_id
JOIN   dim_prodotto cat      ON f.prodotto_id      = cat.prodotto_id
WHERE  f.data_vendita BETWEEN DATE '2025-10-01' AND DATE '2025-12-31'
GROUP BY pv.regione, cat.famiglia
ORDER BY fatturato DESC;
```

Il predicato su `data_vendita` avrebbe dovuto usare l'indice. E in effetti lo usava — un anno prima, quando la tabella aveva 500 milioni di righe. Ma con 800 milioni, l'optimizer aveva deciso che l'indice non conveniva più. Il calcolo era semplice: un trimestre = circa il 8% delle righe totali. Con un index range scan, Oracle avrebbe dovuto fare 64 milioni di accessi random ai blocchi dati. Un {{< glossary term="full-table-scan" >}}full table scan{{< /glossary >}} sequenziale costava meno.

E così faceva: leggeva 800 milioni di righe per restituirne 64 milioni.

```
--------------------------------------------------------------
| Id | Operation          | Name         | Rows  | Bytes    |
--------------------------------------------------------------
|  0 | SELECT STATEMENT   |              |  1200 |    84K   |
|  1 |  SORT GROUP BY     |              |  1200 |    84K   |
|* 2 |   HASH JOIN        |              |   64M | 4480M    |
|  3 |    TABLE ACCESS FULL| DIM_PRODOTTO |  12K  |  168K    |
|* 4 |    HASH JOIN        |              |   64M | 3520M    |
|  5 |     TABLE ACCESS FULL| DIM_PUNTO_VENDITA | 200 | 4000 |
|* 6 |     TABLE ACCESS FULL| FACT_VENDITE| 800M  | 40G      |
--------------------------------------------------------------
```

Quaranta gigabyte di I/O per una query trimestrale. In un ambiente dove il buffer pool era dimensionato a 16 GB, significava leggere più di due volte l'intero cache da disco. Dodici minuti.

## 🏗️ La soluzione: range partitioning per mese

Il partizionamento range per data è la scelta naturale per una fact table in un data warehouse. I dati entrano in ordine cronologico, le query filtrano per periodo, i dati vecchi diventano freddi e quelli nuovi caldi. La data è la chiave di partizionamento perfetta.

Ho scelto il partizionamento mensile — 36 partizioni per tre anni di storico, più una partizione per i dati correnti. Ogni partizione conteneva circa 48 milioni di righe: un volume gestibile per le query e per le operazioni di manutenzione.

```sql
CREATE TABLE fact_vendite_part (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20)
)
PARTITION BY RANGE (data_vendita) (
    PARTITION p_2023_01 VALUES LESS THAN (DATE '2023-02-01'),
    PARTITION p_2023_02 VALUES LESS THAN (DATE '2023-03-01'),
    PARTITION p_2023_03 VALUES LESS THAN (DATE '2023-04-01'),
    -- ... 33 partizioni intermedie ...
    PARTITION p_2025_12 VALUES LESS THAN (DATE '2026-01-01'),
    PARTITION p_max     VALUES LESS THAN (MAXVALUE)
);
```

Con un {{< glossary term="local-index" >}}indice locale{{< /glossary >}} sulla data:

```sql
CREATE INDEX idx_vendite_data_local ON fact_vendite_part (data_vendita) LOCAL;
CREATE INDEX idx_vendite_pv_local   ON fact_vendite_part (punto_vendita_id, data_vendita) LOCAL;
```

Ogni partizione ha il suo segmento di indice. Quando l'optimizer elimina una partizione, elimina anche il segmento di indice corrispondente.

## 📦 La migrazione: da monolite a partizioni

Migrare 800 milioni di righe non è un'operazione che si fa con un semplice INSERT...SELECT. Serve una strategia.

Ho usato l'approccio {{< glossary term="ctas" >}}CTAS{{< /glossary >}} (Create Table As Select) con {{< glossary term="nologging" >}}NOLOGGING{{< /glossary >}} e parallelismo. La procedura era:

1. Creare la tabella partizionata vuota con la struttura definitiva
2. Popolarla con un INSERT diretto dalla tabella originale
3. Ricostruire gli indici
4. Validare i conteggi
5. Rinominare le tabelle (swap)
6. Fare un backup RMAN immediato (NOLOGGING richiede backup)

```sql
ALTER SESSION ENABLE PARALLEL DML;

INSERT /*+ APPEND PARALLEL(8) */ INTO fact_vendite_part
SELECT * FROM fact_vendite;

COMMIT;
```

Con 8 processi paralleli e NOLOGGING, il caricamento ha impiegato 47 minuti per 800 milioni di righe. Non male, considerando che ogni riga doveva essere distribuita nella partizione corretta in base alla data.

Poi la fase di validazione:

```sql
SELECT 'Originale' AS fonte, COUNT(*) AS righe FROM fact_vendite
UNION ALL
SELECT 'Partizionata', COUNT(*) FROM fact_vendite_part;
```

800.247.331 righe da entrambe le parti. Perfetto.

```sql
ALTER TABLE fact_vendite RENAME TO fact_vendite_old;
ALTER TABLE fact_vendite_part RENAME TO fact_vendite;
```

La tabella originale l'ho tenuta per una settimana come rete di sicurezza, poi l'ho droppata.

## ⚡ Il {{< glossary term="partition-pruning" >}}partition pruning{{< /glossary >}} in azione

Con il partizionamento in posizione, la stessa query trimestrale di prima cambiava completamente execution plan:

```sql
SELECT pv.regione,
       cat.famiglia,
       SUM(f.importo)    AS fatturato,
       SUM(f.quantita)   AS pezzi_venduti,
       SUM(f.sconto)     AS sconto_totale
FROM   fact_vendite f
JOIN   dim_punto_vendita pv  ON f.punto_vendita_id = pv.punto_vendita_id
JOIN   dim_prodotto cat      ON f.prodotto_id      = cat.prodotto_id
WHERE  f.data_vendita BETWEEN DATE '2025-10-01' AND DATE '2025-12-31'
GROUP BY pv.regione, cat.famiglia
ORDER BY fatturato DESC;
```

```
----------------------------------------------------------------------
| Id | Operation                   | Name         | Pstart | Pstop  |
----------------------------------------------------------------------
|  0 | SELECT STATEMENT            |              |        |        |
|  1 |  SORT GROUP BY              |              |        |        |
|* 2 |   HASH JOIN                 |              |        |        |
|  3 |    TABLE ACCESS FULL        | DIM_PRODOTTO |        |        |
|* 4 |    HASH JOIN                |              |        |        |
|  5 |     TABLE ACCESS FULL       | DIM_PUNTO_V  |        |        |
|  6 |     PARTITION RANGE ITERATOR|              |  34    |  36    |
|* 7 |      TABLE ACCESS FULL      | FACT_VENDITE |  34    |  36    |
----------------------------------------------------------------------
```

`Pstart: 34, Pstop: 36`. L'optimizer leggeva solo tre partizioni su 37 — ottobre, novembre e dicembre 2025. Invece di 800 milioni di righe, ne scansionava 144 milioni. Invece di 40 GB di I/O, circa 7 GB.

Il risultato? Da 12 minuti a 40 secondi.

Non perché l'hardware fosse più veloce, non perché avessi riscritto le query. Solo perché il database adesso sapeva dove *non* cercare.

## 🔄 Exchange partition: il caricamento che non costa nulla

In un data warehouse, i dati arrivano con una cadenza regolare — nel nostro caso, un {{< glossary term="etl" >}}ETL{{< /glossary >}} notturno che caricava le vendite del giorno. Il problema classico del partizionamento è: come carichi i nuovi dati nella partizione corretta senza impattare le query?

La risposta si chiama {{< glossary term="exchange-partition" >}}exchange partition{{< /glossary >}}.

Il processo funzionava così:

1. L'ETL carica i dati della giornata in una tabella di staging non partizionata
2. Si costruiscono gli indici sulla staging table (stessa struttura degli indici locali)
3. Si esegue l'exchange partition: la staging table e la partizione target si scambiano i segmenti

```sql
-- 1. Tabella di staging con stessa struttura della partizione
CREATE TABLE stg_vendite_daily (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20)
);

-- 2. Caricamento ETL nella staging (operazione indipendente)
INSERT /*+ APPEND */ INTO stg_vendite_daily
SELECT * FROM source_vendite WHERE data_vendita = TRUNC(SYSDATE - 1);

-- 3. Exchange: swap istantaneo dei segmenti
ALTER TABLE fact_vendite
EXCHANGE PARTITION p_2026_01 WITH TABLE stg_vendite_daily
INCLUDING INDEXES WITHOUT VALIDATION;
```

L'exchange partition è un'operazione DDL che modifica solo il data dictionary — non sposta nemmeno un byte di dati. Ci mette meno di un secondo, indipendentemente dal volume. E durante l'exchange, le query sulle altre partizioni continuano a funzionare senza interruzione.

Nel nostro caso, l'ETL notturno accumulava i dati della giornata nella staging, e a fine mese si faceva l'exchange con la partizione del mese corrente. Durante il mese, i dati giornalieri andavano nella partizione `p_max` (la catch-all) e poi venivano consolidati con un exchange mensile.

## 📊 Il ciclo di vita dei dati

Con il partizionamento la gestione del ciclo di vita diventa banale. Dopo tre anni, la partizione più vecchia si può:

- **comprimere**: `ALTER TABLE fact_vendite MODIFY PARTITION p_2023_01 COMPRESS FOR QUERY HIGH;`
- **spostare su storage più lento**: `ALTER TABLE fact_vendite MOVE PARTITION p_2023_01 TABLESPACE ts_archivio;`
- **eliminare direttamente**: `ALTER TABLE fact_vendite DROP PARTITION p_2023_01;`

Droppare una partizione è istantaneo — è un'operazione sul data dictionary, non cancella riga per riga. Confrontalo con un `DELETE FROM fact_vendite WHERE data_vendita < DATE '2023-02-01'` su 48 milioni di righe: minuti di elaborazione, tonnellate di redo log, e una tabella piena di spazio recuperabile che richiede un reorganize.

Nel progetto GDO, la policy era: 3 anni online compressi, poi drop. Ogni primo del mese un job schedulato creava la nuova partizione e, se necessario, droppava quella di 37 mesi prima. Completamente automatico.

## 🎯 Quello che il partizionamento non risolve

Il partizionamento non è una bacchetta magica. Non sostituisce gli indici — se la query non filtra per la chiave di partizionamento, il pruning non si attiva e il database legge tutte le partizioni. Non migliora le query che già usano un indice efficiente su poche righe. E aggiunge complessità nella gestione: partizioni da creare, monitorare, comprimere, eliminare.

Ma per una fact table in un data warehouse — dove i dati sono cronologici, le query filtrano per periodo, e i volumi crescono ogni giorno — il partizionamento range per data non è un'opzione. È un requisito architetturale.

Il collega con il report da 12 minuti non aveva un problema di hardware o di query mal scritte. Aveva una tabella che era cresciuta oltre il punto in cui la mancanza di struttura fisica diventa un collo di bottiglia. Il partizionamento ha rimesso le cose al loro posto: 40 secondi, e nessuna riga letta inutilmente.

------------------------------------------------------------------------

## Glossario

**[Range Partitioning](/it/glossary/range-partitioning/)** — Strategia di partizionamento che divide una tabella in segmenti basati su intervalli di valori di una colonna (tipicamente una data). Ogni partizione contiene le righe il cui valore cade nell'intervallo definito.

**[Exchange Partition](/it/glossary/exchange-partition/)** — Operazione DDL Oracle che scambia istantaneamente i segmenti dati tra una tabella non partizionata e una partizione, senza spostare fisicamente i dati. Usata nei data warehouse per caricamenti bulk a impatto zero.

**[Partition Pruning](/it/glossary/partition-pruning/)** — Meccanismo automatico dell'optimizer Oracle che esclude le partizioni non rilevanti durante l'esecuzione di una query, leggendo solo quelle corrispondenti al predicato WHERE.

**[Fact table](/it/glossary/fact-table/)** — Tabella centrale dello star schema che contiene le misure numeriche del business (importi, quantità, conteggi) e le chiavi esterne verso le tabelle dimensionali.

**[Full Table Scan](/it/glossary/full-table-scan/)** — Operazione di lettura in cui il database percorre tutti i blocchi di una tabella senza utilizzare indici. Efficiente su grandi volumi quando la selettività è bassa, costoso quando si cercano poche righe.
