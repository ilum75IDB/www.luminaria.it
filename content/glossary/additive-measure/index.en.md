---
title: "Additive Measure"
description: "A numeric measure in a fact table that can be summed across all dimensions — amounts, quantities, counts. Fundamental in data warehouse design."
translationKey: "glossary_additive_measure"
aka: "Fully additive measure"
articles:
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

An **additive measure** is a numeric value in a fact table that can be legitimately summed across any dimension: by customer, by product, by period, by territory.

## How it works

Measures in fact tables fall into three categories:

- **Additive**: can be summed across all dimensions (e.g., sales amount, quantity, cost). The most common and most useful
- **Semi-additive**: can be summed across some dimensions but not across time (e.g., account balance: summable by branch, not by month)
- **Non-additive**: cannot be summed at all (e.g., percentages, ratios, pre-calculated averages)

## What it's for

Additive measures are the heart of every fact table because they enable the aggregations that the business requires: totals by period, by region, by product. The key rule: always store atomic values (the detail), never aggregates. From a line-level amount you can derive the monthly total; from a monthly total you cannot reconstruct the individual lines.

## When to use it

When designing a fact table, every measure should be classified as additive, semi-additive, or non-additive. This determines which aggregations are valid in reports and which would produce incorrect results. A common mistake is treating a semi-additive measure (like a balance) as if it were additive — summing monthly balances to get a "total" that has no business meaning.
