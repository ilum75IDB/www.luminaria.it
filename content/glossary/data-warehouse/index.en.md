---
title: "Data Warehouse"
description: "Centralised data collection and historicisation system from diverse sources, designed for analysis and business decision support."
translationKey: "glossary_data-warehouse"
articles:
  - "/posts/project-management/4-milioni-nessun-software"
---

A **Data Warehouse** (DWH) is a data storage system specifically designed for analysis, reporting and business decision support. Unlike operational databases (OLTP), a DWH collects data from multiple sources, transforms it and organises it into structures optimised for analytical queries.

## How it works

Data is extracted from source systems (ERPs, CRMs, business applications), transformed through ETL processes that clean, normalise and enrich it, and finally loaded into the DWH. The typical data model is the star schema: a central fact table with numerical measures linked to dimension tables that describe context (time, customer, product, geography).

## What it's for

A DWH enables answering business questions that operational systems cannot handle: historical trends, comparative analysis across periods, cross-system aggregations, business KPIs. It separates analytical workload from transactional workload, preventing reporting queries from impacting operational application performance.

## When to use it

A DWH is needed when a company needs to integrate data from diverse sources to produce consolidated analyses. Complexity and costs depend on the number of source systems, data volume and required update frequency.
