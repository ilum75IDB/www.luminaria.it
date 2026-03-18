---
title: "MERGE"
description: "A SQL statement that combines INSERT and UPDATE in a single operation. In Oracle also known as upsert."
translationKey: "glossary_merge_sql"
aka: "UPSERT, MERGE INTO"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**MERGE** is a SQL statement that combines INSERT and UPDATE (and optionally DELETE) operations in a single statement. If the record exists it updates it, if it doesn't it inserts it. It's often informally called "upsert".

## Oracle syntax

``` sql
MERGE INTO target_table d
USING source_table s ON (d.key = s.key)
WHEN MATCHED THEN UPDATE SET
    d.field = s.field
WHEN NOT MATCHED THEN INSERT (key, field)
    VALUES (s.key, s.field);
```

## Use in data warehousing

In an ETL context, MERGE is the core mechanism for loading dimension tables:

- **SCD Type 1**: a single MERGE that updates existing records and inserts new ones
- **SCD Type 2**: MERGE is used in the first phase to close modified records (setting the end validity date), followed by an INSERT for the new versions

## Availability

- **Oracle**: full support since version 9i
- **PostgreSQL**: no native MERGE until version 15. The alternative is `INSERT ... ON CONFLICT` (upsert)
- **MySQL**: uses `INSERT ... ON DUPLICATE KEY UPDATE` as an alternative
- **SQL Server**: full support with syntax similar to Oracle
