---
title: "Star schema"
description: "A data model typical of data warehouses: a central fact table connected to multiple dimension tables via foreign keys."
translationKey: "glossary_star_schema"
aka: "Schema a stella"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

A **star schema** is the most widely used data model in data warehousing. It gets its name from its shape: a central fact table connected to multiple dimension tables surrounding it, like the points of a star.

## Structure

- **Fact table** at the center: contains numeric measures and foreign keys to dimensions
- **Dimension tables** around it: contain descriptive attributes (who, what, where, when) in a denormalized structure

Dimensions in a star schema are typically denormalized — all attributes in a single flat table, with no normalized hierarchies. This simplifies queries and improves aggregation performance.

## Why it works

The star schema is optimized for analytical queries:

- Joins are simple: the fact connects directly to each dimension with a single join
- Aggregations are fast: database optimizers recognize the pattern and optimize for it
- It's intuitive for business users: the structure mirrors how they think about data (sales by product, by region, by period)

## Star schema vs Snowflake

A **snowflake schema** normalizes the dimensions, splitting them into sub-tables. It saves space but complicates queries with additional joins. In practice, star schemas are preferred in most cases because the simplicity of queries far outweighs the cost of extra space in dimensions.
