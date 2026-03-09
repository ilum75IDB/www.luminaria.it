---
title: "Oracle Partitioning: when 2 billion rows no longer fit in a query"
description: "A client with a 2-billion-row transaction table and reporting queries that had gone from seconds to hours. How I solved it with Oracle partitioning — range, interval, partition pruning and local indexes."
date: "2025-12-23T10:00:00+01:00"
draft: false
translationKey: "oracle_partitioning"
tags: ["partitioning", "performance", "tuning", "execution-plan"]
categories: ["oracle"]
image: "oracle-partitioning.cover.jpg"
---

Two billion rows. You do not reach that number in a day. It takes years of transactions, movements, daily records piling up. And for all that time the database works, queries respond, reports come out. Then one day someone opens a ticket: "the monthly report takes four hours."

Four hours. For a report that six months earlier took twenty minutes.

It is not a bug. It is not a network issue or slow storage. It is the physics of data: when a table grows beyond a certain threshold, the approaches that worked stop working. And if you did not design the structure to handle that growth, the database does the only thing it can: read everything.

---

## The context: telecoms and industrial volumes

The client was a telecom operator. Nothing exotic — a classic Oracle 19c Enterprise Edition environment on Linux, SAN storage, about thirty instances across production, staging and development. The critical instance was billing: invoicing, CDR (Call Detail Records), accounting movements.

The table at the centre of the problem was called `TXN_MOVIMENTI`. It collected every single transaction from the billing system since 2016. The structure was roughly this:

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

2.1 billion rows. 380 GB of data. A single segment, a single tablespace, no partitions. A monolith.

The indexes were there: one on the primary key, one on `data_movimento`, one composite on `(cod_cliente, data_movimento)`. But when a table exceeds a certain size, even an index range scan is no longer enough, because the volume of data returned is still enormous.

---

## The symptoms: it is not slowness, it is collapse

The problems did not show up all at once. They arrived gradually, as always happens with tables that grow without control.

**First signal**: monthly reports. The aggregate billing query — summing amounts by customer for a given month — had gone from 20 minutes to 4 hours over the course of a year. The execution plan showed an index range scan on the date, but the number of blocks read was monstrous: Oracle had to traverse hundreds of thousands of index leaf blocks and then do table access by rowid to retrieve the columns not covered by the index.

**Second signal**: maintenance. `ALTER INDEX REBUILD` on the date index took six hours. Statistics collection (`DBMS_STATS.GATHER_TABLE_STATS`) would not finish overnight. RMAN backups had become a gamble: sometimes they fit in the window, sometimes not.

**Third signal**: involuntary full table scans. Queries with date predicates that the optimizer chose to resolve with a full table scan because the estimated cost of the index scan was higher. On 380 GB of data.

The execution plan for the billing query looked like this:

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

28 million rows for January alone. The index found the rows, but then Oracle had to fetch each individual row from the table to read `cod_cliente`, `importo` and `stato`. Millions of random I/O operations on a 380 GB table scattered across thousands of blocks.

---

## The solution: you do not need a better index, you need a different structure

I spent two days analysing access patterns before proposing any solution. Because partitioning is not a magic wand — if you get the partition key wrong, you make things worse.

The patterns were clear:

- **90% of queries** had a predicate on the date (`data_movimento`)
- Reports were always **monthly or quarterly**
- Operational queries (single customer) always used `cod_cliente + data_movimento`
- Data older than 3 years was never read by reports, only by annual archival batches

The choice fell on **monthly interval partitioning** on the `data_movimento` column. Not classic range partitioning, where you have to manually create each future partition. Interval: you define the interval once and Oracle creates partitions automatically when data arrives for a new period.

---

## The implementation: CTAS, local indexes and zero downtime (almost)

You cannot do `ALTER TABLE ... PARTITION BY` on an existing table with 2 billion rows. Not in Oracle 19c, at least not without Online Table Redefinition. And that option, on a table this size, has its own risks.

I chose the CTAS approach — Create Table As Select — with parallelism. Create the new partitioned table, copy the data, rename.

### Step 1: create the partitioned table

``` sql
CREATE TABLE txn_movimenti_part
PARTITION BY RANGE (data_movimento)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    PARTITION p_before_2016 VALUES LESS THAN (DATE '2016-01-01'),
    PARTITION p_2016_01     VALUES LESS THAN (DATE '2016-02-01'),
    PARTITION p_2016_02     VALUES LESS THAN (DATE '2016-03-01')
    -- Oracle will automatically create subsequent partitions
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

`NOLOGGING` is essential: without it the copy generates redo log for every block written. On 380 GB that would mean filling the redo area and putting the system into archivelog mode for days. With `NOLOGGING` the copy took 3 and a half hours with parallelism at 8.

After the copy I restored logging:

``` sql
ALTER TABLE txn_movimenti_part LOGGING;
```

And ran an RMAN backup immediately, because NOLOGGING segments are not recoverable in case of a restore.

### Step 2: local indexes

Index design on a partitioned table is different from a regular table. The key concept is: **local index vs global index**.

A **local** index is partitioned with the same key as the table. Each table partition has its corresponding index partition. Advantage: maintenance operations on one partition do not touch the others.

A **global** index spans all partitions. It is more efficient for queries that do not filter on the partition key, but any DDL operation on the partition (drop, truncate, split) invalidates the entire index.

``` sql
-- Primary key as global index (needed for point lookups)
ALTER TABLE txn_movimenti_part
ADD CONSTRAINT pk_txn_mov_part PRIMARY KEY (txn_id)
USING INDEX GLOBAL;

-- Local index on date (partition-aligned)
CREATE INDEX idx_txn_mov_data ON txn_movimenti_part (data_movimento)
LOCAL PARALLEL 8;

-- Local composite index for operational queries
CREATE INDEX idx_txn_mov_cli_data
ON txn_movimenti_part (cod_cliente, data_movimento)
LOCAL PARALLEL 8;
```

The primary key stays global because queries by `txn_id` never include the date — you need direct access. The other indexes are local because they align with usage patterns: queries by date, queries by customer+date.

### Step 3: the switch

``` sql
-- Rename the original table (backup)
ALTER TABLE txn_movimenti RENAME TO txn_movimenti_old;

-- Rename the new table
ALTER TABLE txn_movimenti_part RENAME TO txn_movimenti;

-- Rebuild synonyms if any
-- Recompile invalidated objects
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

The actual downtime was the time for the two `ALTER TABLE RENAME` statements: a few seconds. Everything else — data copy, index creation — happened in parallel while the system was live.

### Step 4: gather statistics

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

The `granularity => 'ALL'` parameter is important: it tells Oracle to gather statistics at the global, partition and subpartition level. Without it, the optimizer might make wrong decisions because it does not know the data distribution within individual partitions.

---

## Before and after: the numbers

The same billing query, after partitioning:

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

Look at step 2: `PARTITION RANGE SINGLE`. Oracle knows that January data sits in a single partition and reads only that one. The full table scan that used to be terrifying is now a full **partition** scan — on about 4 GB instead of 380.

| Metric | Before | After | Change |
|---|---|---|---|
| Monthly query time | 4 hours | 3 minutes | -98% |
| Consistent gets | 48M | 580K | -98.8% |
| Physical reads | 12M | 95K | -99.2% |
| GATHER_TABLE_STATS time | 14 hours | 25 min (per partition) | -97% |
| Index rebuild time | 6 hours | 12 min (per partition) | -97% |
| Incremental backup size | 380 GB | ~4 GB/month | -99% |

The cost went from 890K to 12K. That is not a percentage improvement — it is a different order of magnitude.

---

## Partition pruning: the real magic

The mechanism that makes all this possible is called **partition pruning**. It is not something you configure — Oracle does it automatically when the query predicate matches the partition key.

But you need to know when it works and when it does not.

**It works** with direct predicates on the partition column:

``` sql
-- Pruning active: Oracle reads only the January partition
WHERE data_movimento BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'

-- Pruning active: Oracle reads only the specific partition
WHERE data_movimento = DATE '2025-03-15'
```

**It does not work** when the column is wrapped in a function:

``` sql
-- Pruning DISABLED: Oracle must read all partitions
WHERE TRUNC(data_movimento) = DATE '2025-01-01'

-- Pruning DISABLED: function on the column
WHERE TO_CHAR(data_movimento, 'YYYY-MM') = '2025-01'

-- Pruning DISABLED: arithmetic expression
WHERE data_movimento + 30 > SYSDATE
```

This is the most common mistake I see after a partitioning implementation: developers apply functions to the date column without realising they are disabling pruning. And the table goes back to being read in full.

I spent half a day reviewing every application query that touched `TXN_MOVIMENTI`. I found eleven with `TRUNC(data_movimento)` in the `WHERE` clause. Eleven queries that would have ignored the partitioning.

---

## Lifecycle management: drop partition

One of the most concrete advantages of partitioning is data lifecycle management. Before partitioning, archiving old data meant a `DELETE` of billions of rows — an operation that generates mountains of redo and undo, locks the table for hours and risks blowing up the undo tablespace.

With partitioning:

``` sql
-- Archive 2016 data to a read-only tablespace
ALTER TABLE txn_movimenti
MOVE PARTITION p_2016_01 TABLESPACE ts_archive;

-- Or, if the data is no longer needed
ALTER TABLE txn_movimenti DROP PARTITION p_2016_01;
```

A `DROP PARTITION` on a 4 GB partition takes less than a second. It generates no undo. It generates no significant redo. It does not lock the other partitions. It is a DDL operation, not DML.

I set up a monthly job that moved partitions older than 5 years to the archive tablespace and set them to read-only. The client recovered 120 GB of active space without deleting a single record.

---

## What I learned (and the mistakes to avoid)

After fifteen years of Oracle partitioning, I have a list of things I wish I had known earlier.

**The partition key must match the access pattern.** It sounds obvious, but I have seen tables partitioned by `cod_cliente` when 95% of queries filter by date. Partitioning only works if queries can prune.

**Interval partitioning is almost always better than static range.** With classic range you have to manually create future partitions, which means a scheduled job or a DBA who remembers. With interval Oracle creates them on its own. One less problem.

**Global indexes are a trap.** They work well for queries, but any DDL operation on the partition invalidates them. And rebuilding a global index on 2 billion rows takes hours. Use local indexes where possible and accept the trade-off.

**NOLOGGING is not optional for bulk operations.** Without NOLOGGING, a 380 GB CTAS generates the same amount of redo. Your archivelog area will fill up, the database will go into wait, and the on-call DBA will get a phone call at 3 in the morning.

**Test pruning before going to production.** Do not trust: verify with `EXPLAIN PLAN` that every critical query actually prunes. A single `TRUNC()` in the wrong predicate and you have a 380 GB full table scan.

**Partitioning does not replace indexes.** It reduces the volume of data to examine, but inside the partition you still need the right indexes. A monthly partition of 28 million rows without an index is still a problem.

---

## When you need partitioning

Not every table needs partitioning. My rule of thumb:

- Under 10 million rows: probably not
- Between 10 and 100 million: depends on access patterns and growth rate
- Over 100 million: probably yes
- Over a billion: you have no choice

But the right time to implement it is before it becomes urgent. When the table already has 2 billion rows, the migration is a project in itself. When it has 50 million and is growing, it is an afternoon's work.

My biggest mistake with partitioning? Not proposing it six months earlier, when all the signals were already there.
