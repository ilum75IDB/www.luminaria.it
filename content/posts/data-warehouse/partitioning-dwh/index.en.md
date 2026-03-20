---
title: "DWH Partitioning: When 3 Years of Data Weigh Too Much"
description: "An 800-million-row fact table with no partitioning, quarterly queries running for 12 minutes, and a business demanding real-time answers. How I implemented monthly range partitioning and brought query times down to 40 seconds."
date: "2026-04-07T10:00:00+01:00"
draft: false
translationKey: "partitioning_dwh"
tags: ["partitioning", "performance", "oracle", "fact-table", "data-warehouse"]
categories: ["data-warehouse"]
image: "partitioning-dwh.cover.jpg"
---

Last week a colleague told me about a project where data warehouse queries had stopped returning in any reasonable time. "How long does the quarterly report take?" I asked. "Twelve minutes." "And before?" "A minute and a half."

I didn't need to ask anything else. I already knew the script.

A {{< glossary term="fact-table" >}}fact table{{< /glossary >}} that starts small, grows every day, and nobody worries about the physical structure until one day queries stop coming back. It's not a bug, not a code error. It's the weight of data finally making itself felt.

---

## The context: retail and three years of receipts

The project was in the grocery retail sector — a supermarket chain with around two hundred stores, roughly a hundred million euros in annual revenue, and an Oracle 19c data warehouse collecting everything: sales, returns, warehouse movements, promotions.

The table at the center of the problem was called `FACT_VENDITE`. Each row was a receipt line — an average receipt has eight lines, multiplied by thirty thousand receipts per day across two hundred stores, that's about 48 million rows per month. Over three years, 800 million rows had piled up.

The structure looked like this:

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

A single index on the primary key, an index on `data_vendita`, and a composite one on `(punto_vendita_id, data_vendita)`. No partitioning. Eight hundred million rows in a single monolithic table.

## 🔍 The symptom: full table scan on 800 million rows

The DWH's analytical queries almost always worked by time period. Last quarter sales by store. Year-over-year comparison by product category. Monthly margins by region. All queries with a filter on `data_vendita`.

The quarterly report was this:

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

The predicate on `data_vendita` should have used the index. And it did — a year earlier, when the table had 500 million rows. But with 800 million, the optimizer had decided the index was no longer worth it. The math was straightforward: one quarter = roughly 8% of total rows. With an index range scan, Oracle would have needed 64 million random block accesses. A sequential {{< glossary term="full-table-scan" >}}full table scan{{< /glossary >}} cost less.

And so it did: it read 800 million rows to return 64 million.

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

Forty gigabytes of I/O for a quarterly query. In an environment where the buffer pool was sized at 16 GB, that meant reading more than twice the entire cache from disk. Twelve minutes.

## 🏗️ The solution: monthly range partitioning

Range partitioning by date is the natural choice for a fact table in a data warehouse. Data enters in chronological order, queries filter by time period, old data goes cold and new data stays hot. The date is the perfect partition key.

I chose monthly partitioning — 36 partitions for three years of history, plus one partition for current data. Each partition held about 48 million rows: a manageable volume for queries and maintenance operations.

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
    -- ... 33 intermediate partitions ...
    PARTITION p_2025_12 VALUES LESS THAN (DATE '2026-01-01'),
    PARTITION p_max     VALUES LESS THAN (MAXVALUE)
);
```

With a {{< glossary term="local-index" >}}local index{{< /glossary >}} on the date:

```sql
CREATE INDEX idx_vendite_data_local ON fact_vendite_part (data_vendita) LOCAL;
CREATE INDEX idx_vendite_pv_local   ON fact_vendite_part (punto_vendita_id, data_vendita) LOCAL;
```

Each partition has its own index segment. When the optimizer eliminates a partition, it eliminates the corresponding index segment too.

## 📦 The migration: from monolith to partitions

Migrating 800 million rows is not something you do with a simple INSERT...SELECT. You need a strategy.

I used the {{< glossary term="ctas" >}}CTAS{{< /glossary >}} (Create Table As Select) approach with {{< glossary term="nologging" >}}NOLOGGING{{< /glossary >}} and parallelism. The procedure was:

1. Create the empty partitioned table with the final structure
2. Populate it with a direct-path INSERT from the original table
3. Rebuild the indexes
4. Validate the row counts
5. Rename the tables (swap)
6. Run an immediate RMAN backup (NOLOGGING requires a backup)

```sql
ALTER SESSION ENABLE PARALLEL DML;

INSERT /*+ APPEND PARALLEL(8) */ INTO fact_vendite_part
SELECT * FROM fact_vendite;

COMMIT;
```

With 8 parallel processes and NOLOGGING, the load took 47 minutes for 800 million rows. Not bad, considering each row had to be routed to the correct partition based on its date.

Then the validation phase:

```sql
SELECT 'Original' AS source, COUNT(*) AS rows FROM fact_vendite
UNION ALL
SELECT 'Partitioned', COUNT(*) FROM fact_vendite_part;
```

800,247,331 rows on both sides. Perfect.

```sql
ALTER TABLE fact_vendite RENAME TO fact_vendite_old;
ALTER TABLE fact_vendite_part RENAME TO fact_vendite;
```

I kept the original table for a week as a safety net, then dropped it.

## ⚡ {{< glossary term="partition-pruning" >}}Partition pruning{{< /glossary >}} in action

With partitioning in place, the same quarterly query had a completely different execution plan:

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

`Pstart: 34, Pstop: 36`. The optimizer read only three partitions out of 37 — October, November, and December 2025. Instead of 800 million rows, it scanned 144 million. Instead of 40 GB of I/O, about 7 GB.

The result? From 12 minutes to 40 seconds.

Not because the hardware was faster, not because I rewrote the queries. Only because the database now knew where *not* to look.

## 🔄 Exchange partition: the zero-cost load

In a data warehouse, data arrives on a regular cadence — in our case, a nightly {{< glossary term="etl" >}}ETL{{< /glossary >}} that loaded each day's sales. The classic partitioning challenge is: how do you load new data into the correct partition without impacting queries?

The answer is {{< glossary term="exchange-partition" >}}exchange partition{{< /glossary >}}.

The process worked like this:

1. The ETL loads the day's data into a non-partitioned staging table
2. Build indexes on the staging table (same structure as the local indexes)
3. Execute the exchange partition: the staging table and the target partition swap their segments

```sql
-- 1. Staging table with same structure as a partition
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

-- 2. ETL load into staging (independent operation)
INSERT /*+ APPEND */ INTO stg_vendite_daily
SELECT * FROM source_vendite WHERE data_vendita = TRUNC(SYSDATE - 1);

-- 3. Exchange: instantaneous segment swap
ALTER TABLE fact_vendite
EXCHANGE PARTITION p_2026_01 WITH TABLE stg_vendite_daily
INCLUDING INDEXES WITHOUT VALIDATION;
```

The exchange partition is a DDL operation that only modifies the data dictionary — it doesn't move a single byte of data. It takes less than a second, regardless of volume. And during the exchange, queries on other partitions continue working without interruption.

In our case, the nightly ETL accumulated each day's data in the staging table, and at month-end we did the exchange with the current month's partition. During the month, daily data went into the `p_max` partition (the catch-all) and was then consolidated with a monthly exchange.

## 📊 Data lifecycle management

With partitioning, lifecycle management becomes trivial. After three years, the oldest partition can be:

- **compressed**: `ALTER TABLE fact_vendite MODIFY PARTITION p_2023_01 COMPRESS FOR QUERY HIGH;`
- **moved to slower storage**: `ALTER TABLE fact_vendite MOVE PARTITION p_2023_01 TABLESPACE ts_archivio;`
- **dropped entirely**: `ALTER TABLE fact_vendite DROP PARTITION p_2023_01;`

Dropping a partition is instantaneous — it's a data dictionary operation, not a row-by-row delete. Compare that with a `DELETE FROM fact_vendite WHERE data_vendita < DATE '2023-02-01'` on 48 million rows: minutes of processing, tons of redo log, and a table full of reclaimable space that needs a reorganize.

In the retail project, the policy was: 3 years online compressed, then drop. Every first of the month, a scheduled job created the new partition and, if needed, dropped the one from 37 months ago. Fully automatic.

## 🎯 What partitioning doesn't solve

Partitioning is not a magic wand. It doesn't replace indexes — if the query doesn't filter on the partition key, pruning doesn't kick in and the database reads all partitions. It doesn't improve queries that already use an efficient index on a few rows. And it adds management complexity: partitions to create, monitor, compress, and drop.

But for a fact table in a data warehouse — where data is chronological, queries filter by time period, and volumes grow every day — range partitioning by date is not optional. It's an architectural requirement.

The colleague with the 12-minute report didn't have a hardware problem or badly written queries. He had a table that had grown past the point where the lack of physical structure becomes a bottleneck. Partitioning put things back in their place: 40 seconds, and not a single row read needlessly.

------------------------------------------------------------------------

## Glossary

**[Range Partitioning](/en/glossary/range-partitioning/)** — A partitioning strategy that divides a table into segments based on value ranges of a column (typically a date). Each partition holds rows whose value falls within the defined range.

**[Exchange Partition](/en/glossary/exchange-partition/)** — An Oracle DDL operation that instantly swaps data segments between a non-partitioned table and a partition, without physically moving any data. Used in data warehouses for zero-impact bulk loads.

**[Partition Pruning](/en/glossary/partition-pruning/)** — An automatic Oracle optimizer mechanism that excludes irrelevant partitions during query execution, reading only those matching the WHERE predicate.

**[Fact table](/en/glossary/fact-table/)** — The central table in a star schema containing the business's numeric measures (amounts, quantities, counts) and foreign keys to dimensional tables.

**[Full Table Scan](/en/glossary/full-table-scan/)** — A read operation where the database traverses every block of a table without using indexes. Efficient on large volumes when selectivity is low, costly when searching for few rows.
