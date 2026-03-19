---
title: "Fact table"
description: "The central table in a star schema containing numeric measures and foreign keys to dimension tables."
translationKey: "glossary_fact_table"
aka: "Tabella dei fatti"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

A **fact table** is the central table of a star schema in a data warehouse. It contains numeric measures — amounts, quantities, counts, durations — and the foreign keys that connect it to dimension tables.

## Structure

Each row in a fact table represents a business event or transaction: a sale, a claim, a shipment, a login. Columns fall into two categories:

- **Foreign keys**: point to dimension tables (who, what, where, when)
- **Measures**: numeric values to aggregate (amount, quantity, margin)

## Types of fact tables

- **Transaction fact**: one row per event (e.g. each sale)
- **Periodic snapshot**: one row per period per entity (e.g. monthly balance per account)
- **Accumulating snapshot**: one row per process, updated at each milestone (e.g. order-shipment-invoice cycle)

## Relationship with SCDs

When dimensions use SCD Type 2, the fact table points to the dimension's surrogate key — not the natural key. This ensures every fact is associated with the correct dimension version for the moment it occurred.
