---
title: "ANALYZE"
description: "The PostgreSQL command that updates table statistics used by the optimizer to choose the execution plan."
translationKey: "glossary_postgresql_analyze"
aka: "ANALYZE (PostgreSQL)"
articles:
  - "/posts/postgresql/explain-analyze-postgresql"
---

**ANALYZE** is the PostgreSQL command that collects statistics about data distribution in tables and stores them in the `pg_statistic` catalog (readable through the `pg_stats` view). The optimizer uses these statistics to estimate cardinality — how many rows each operation will return — and choose the most efficient execution plan.

## What it collects

The statistics collected by ANALYZE include:

- **Most common values**: the most frequent values for each column and their percentage
- **Distribution histograms**: how the remaining values are distributed
- **Number of distinct values**: how many unique values each column has
- **NULL percentage**: how many rows have NULL for each column

The quality of these statistics depends on the number of samples collected, controlled by the `default_statistics_target` parameter.

## Why it matters

Without up-to-date statistics, the optimizer is forced to guess. Wrong estimates lead to disastrous execution plans — such as choosing a nested loop on millions of rows thinking there are only hundreds, or ignoring a perfectly suitable index.

## When to run it

PostgreSQL runs ANALYZE automatically through autovacuum, but the default threshold (50 rows + 10% of live rows) can be too high for rapidly growing tables. Situations where a manual ANALYZE is needed:

- After bulk imports or bulk loads
- After significant changes in data distribution
- When `EXPLAIN ANALYZE` shows cardinality estimates far from actual rows
- After modifying a column's `default_statistics_target`
