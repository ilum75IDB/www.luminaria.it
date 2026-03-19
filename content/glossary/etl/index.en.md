---
title: "ETL"
description: "Extract, Transform, Load — the process of extracting, transforming and loading data from source systems into the data warehouse."
translationKey: "glossary_etl"
aka: "Extract, Transform, Load"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/ragged-hierarchies"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

**ETL** (Extract, Transform, Load) is the fundamental process through which data is moved from source systems (operational databases, files, APIs) into the data warehouse.

## The three phases

- **Extract**: pulling data from source systems. Can be full (complete load) or incremental (only new or changed data)
- **Transform**: cleaning, validating, standardizing and enriching the data. This is where business rules are applied, dimension lookups performed, derived calculations computed
- **Load**: loading the transformed data into the data warehouse tables (fact and dimension)

## Why it matters

ETL is the least visible but most critical part of a data warehouse. If data is extracted incompletely, transformed with incorrect rules, or loaded without checks, everything built on top — reports, dashboards, decisions — will be wrong.

A well-designed ETL also determines the loading window: how long it takes to refresh the data warehouse. In real-world environments, going from 4 hours to 25 minutes can mean the difference between data being current by morning or by afternoon.

## ELT vs ETL

With the rise of cloud data warehouses and high-performance columnar engines, the **ELT** (Extract, Load, Transform) pattern has become common: data is loaded raw into the warehouse and transformed there, leveraging the SQL engine's processing power. The core concept remains the same — what changes is where the transformation happens.
