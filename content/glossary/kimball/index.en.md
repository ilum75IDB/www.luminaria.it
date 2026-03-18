---
title: "Kimball"
description: "Ralph Kimball — data warehouse design methodology based on dimensional modeling, star schemas and bottom-up ETL processes."
translationKey: "glossary_kimball"
aka: "Kimball methodology, Dimensional Modeling"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**Kimball** refers to Ralph Kimball and his data warehouse design methodology, described in *The Data Warehouse Toolkit* (first edition 1996, third edition 2013).

## The approach

The Kimball methodology rests on three pillars:

- **Dimensional modeling**: organizing data into star schemas with fact tables and dimension tables, optimized for analytical queries
- **Bottom-up**: building the DWH starting from individual departmental data marts, progressively integrating them through conformed dimensions
- **Bus architecture**: a framework for ensuring consistency across data marts through shared dimensions and facts

## Slowly Changing Dimensions

Kimball defined the SCD (Slowly Changing Dimension) classification into types 0 through 7, which has become the de facto industry standard. Type 2 — with surrogate keys and validity dates — is the most widely used for tracking dimension history.

## Kimball vs Inmon

The main alternative is Bill Inmon's methodology, which proposes a top-down approach with a normalized (3NF) enterprise data warehouse from which data marts are derived. The two methodologies are not mutually exclusive and many real-world projects adopt elements of both.
