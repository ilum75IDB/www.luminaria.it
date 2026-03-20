---
title: "Range Partitioning"
description: "A partitioning strategy that divides a table into segments based on value ranges of a column, typically a date."
translationKey: "glossary_range-partitioning"
aka: "Range-based partitioning"
articles:
  - "/posts/data-warehouse/partitioning-dwh"
  - "/posts/oracle/oracle-partitioning"
---

**Range Partitioning** is a table partitioning strategy where rows are distributed across different partitions based on the value of a column relative to predefined ranges. The partition column is almost always a date in data warehouses.

## How it works

Each partition is defined with a `VALUES LESS THAN` clause that sets the upper bound of the range. Oracle automatically assigns each row to the correct partition based on the partition column value. If a row has `data_vendita = '2025-03-15'`, it gets inserted into the partition whose range includes that date.

## When to use it

Range partitioning is the natural choice when data has a dominant time dimension — fact tables in data warehouses, log tables, transaction tables. The partition granularity (daily, monthly, quarterly) depends on insert volume and query patterns: partitions that are too small create management overhead, too large and they reduce partition pruning effectiveness.

## Operational advantages

Beyond query performance, range partitioning enables data lifecycle management operations that are impossible on monolithic tables: instant partition drops (no DELETE needed), selective compression of historical partitions, movement to different storage tiers (ILM — Information Lifecycle Management), and exchange partition for zero-impact bulk loads.
