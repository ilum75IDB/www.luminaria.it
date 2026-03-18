---
title: "default_statistics_target"
description: "The PostgreSQL parameter that controls how many samples ANALYZE collects to estimate data distribution in each column."
translationKey: "glossary_postgresql_default_statistics_target"
aka: "default_statistics_target (PostgreSQL)"
articles:
  - "/posts/postgresql/explain-analyze-postgresql"
---

**default_statistics_target** is the PostgreSQL parameter that defines the number of samples collected by the `ANALYZE` command to build statistics for each column. The default value is 100.

## How it works

PostgreSQL samples a certain number of values for each column and uses them to build two structures:

- **Most common values (MCV)**: the list of the most frequent values, with their respective frequencies
- **Histogram**: the distribution of the remaining values, divided into equal-population buckets

The `default_statistics_target` parameter determines how many elements these structures will have. With the default value of 100, the histogram will have 100 buckets and the MCV list will contain up to 100 values.

## When to increase it

For small tables or tables with uniform distribution, 100 samples are sufficient. For large tables with skewed distribution — where a few values dominate most rows — 100 samples can give a distorted picture, leading the optimizer to wrong cardinality estimates.

You can increase the target at the column level:

    ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
    ANALYZE orders;

Values between 500 and 1000 significantly improve estimate quality on columns with non-uniform distribution.

## Practical limits

Beyond 1000 the benefit is marginal and `ANALYZE` itself becomes slower, because it needs to sample more rows and build larger structures. It's a fine-tuning adjustment: apply it only to columns that actually cause wrong estimates, not to every column in every table.
