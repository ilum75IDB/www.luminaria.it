---
title: "Full Table Scan"
description: "A read operation where Oracle reads every block of a table from first to last, without using any index."
translationKey: "glossary_full_table_scan"
aka: "TABLE ACCESS FULL"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Full Table Scan** (or TABLE ACCESS FULL) is an operation where the database reads every data block of a table, from start to finish, without going through any index.

## How it works

Oracle requests blocks from disk (or cache) sequentially, using multi-block reads (`db file scattered read`). Every row in the table is examined, regardless of whether it matches the query criteria.

## When it's a problem

A full table scan on a large table is often a sign of a missing index, stale statistics, or a changed execution plan. In the AWR report it shows up as `db file scattered read` in the Top Wait Events section, with a high percentage of DB time.

## When it's legitimate

On small tables (a few thousand rows) or when the query genuinely needs to read most of the data, a full table scan can be more efficient than an index access. The problem arises when Oracle chooses it on tables with millions of rows to extract just a few records.

## How to identify it

In the execution plan (`EXPLAIN PLAN` or `DBMS_XPLAN`) it appears as a `TABLE ACCESS FULL` operation. In AWR/ASH wait events it manifests as a dominant `db file scattered read`.
