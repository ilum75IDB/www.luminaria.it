---
title: "Partition Pruning"
description: "Automatic Oracle mechanism that excludes irrelevant partitions during query execution, reading only partitions containing data matching the predicate."
translationKey: "glossary_partition-pruning"
articles:
  - "/posts/oracle/oracle-partitioning"
  - "/posts/data-warehouse/partitioning-dwh"
---

**Partition Pruning** is the mechanism by which Oracle, during query execution on a partitioned table, automatically identifies and excludes partitions that cannot contain data relevant to the query predicate.

## How it works

When a query includes a predicate on the partition column (e.g. `WHERE data_movimento BETWEEN ...`), Oracle consults the partition metadata and determines which partitions contain data in the requested range. Only those partitions are read. In the execution plan it appears as `PARTITION RANGE SINGLE` or `PARTITION RANGE ITERATOR`.

## What it's for

On a 380 GB table with monthly partitions, a query for a single month reads only ~4 GB instead of the entire table. Pruning transforms a nightmare full table scan into a manageable full partition scan, reducing I/O by 99%.

## When to use it

Pruning is automatic, but only works with direct predicates on the partition column. Applying functions to the column (`TRUNC(date)`, `TO_CHAR(date)`) disables pruning and forces Oracle to read all partitions. Always verify with `EXPLAIN PLAN` that pruning is active.
