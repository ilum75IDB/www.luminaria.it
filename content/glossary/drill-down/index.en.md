---
title: "Drill-down"
description: "Navigation in reports from an aggregated level to a detail level, typical of OLAP analysis and data warehouses."
translationKey: "glossary_drill_down"
aka: "Hierarchical navigation, Progressive detail"
articles:
  - "/posts/data-warehouse/ragged-hierarchies"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

**Drill-down** is a report navigation operation that allows moving from an aggregated level to a more detailed level, descending through a hierarchy.

## How it works

In a Top Group → Group → Client hierarchy:

1. Start at the highest level: total revenue by Top Group
2. Click on a Top Group to see its Groups (first-level drill-down)
3. Click on a Group to see individual Clients (second-level drill-down)

The reverse operation — going back up from detail to aggregate — is called **drill-up** (or roll-up).

## Requirements for correct drill-down

To work without errors, drill-down requires:

- A **complete** hierarchy: no missing levels (no NULLs)
- **Total consistency**: the sum of values at the detail level must match the total at the higher level
- **Balanced structure**: all branches of the hierarchy must have the same depth

If the hierarchy is unbalanced (ragged hierarchy), drill-down produces incomplete or incorrect results. Self-parenting solves this by balancing the structure upstream.

## Drill-down vs filter

Drill-down is not a simple filter: it's structured navigation along a predefined hierarchy. A filter shows a subset of data; a drill-down shows the next level of detail within a hierarchical context.
