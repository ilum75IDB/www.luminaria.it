---
title: "Grain"
description: "The level of detail of a fact table in a data warehouse — the design decision that determines which questions the dimensional model can answer."
translationKey: "glossary_grain"
aka: "Granularity, Level of detail"
articles:
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

The **grain** (granularity) is the level of detail of a fact table in a data warehouse. It defines what a single row represents: a transaction, a daily summary, a monthly total, an invoice line.

## How it works

Choosing the grain is the first decision when designing a fact table. Every other choice — measures, dimensions, ETL — follows from it:

- **Fine grain** (e.g., invoice line): maximum query flexibility, more rows to manage
- **Aggregated grain** (e.g., monthly total per customer): fewer rows, faster queries, but no ability to drill into detail

Kimball's fundamental principle: always model at the finest level of detail available in the source system.

## What it's for

The grain determines:

- Which **questions** the data warehouse can answer
- Which **dimensions** are needed (a line-level grain requires dim_product, a monthly grain doesn't)
- How large the fact table is and how long the ETL takes
- Whether **drill-down** in reports is possible or not

## When to use it

The grain is defined during the dimensional model design phase, before writing any DDL or ETL. Changing the grain after go-live is equivalent to rebuilding the data warehouse from scratch — which is why the initial choice is so critical.
