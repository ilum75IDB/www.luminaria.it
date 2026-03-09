---
title: "When a LIKE '%value%' Slows Everything Down: A Real PostgreSQL Optimization Case"
description: "A real-world PostgreSQL performance case where a LIKE '%value%' caused a full scan and degraded response times. Analysis, execution plan, and a scalable indexing strategy."
date: "2026-01-06T10:00:00+01:00"
lastmod: "2026-02-26T09:34:00+01:00"
draft: false
translationKey: "like_optimization_postgresql"
tags: ["query-tuning", "performance", "indexes", "pg_trgm"]
categories: ["postgresql"]
image: "like-optimization-postgresql.cover.jpg"
---

A few weeks ago, a client contacted me with a very common issue:

> "Search in the admin console is slow. Sometimes it takes several
> seconds. We've already reduced the JOINs, but the problem hasn't
> disappeared."

Environment: PostgreSQL on managed cloud.\
Main table: `payment_report` (\~6 million rows, 3 GB).\
Searched column: `reference_code`.

Problematic query:

``` sql
SELECT *
FROM reporting.payment_report r
JOIN reporting.payment_cart c ON c.id = r.cart_id
WHERE c.service_id = 1001
  AND r.reference_code LIKE '%ABC123%'
ORDER BY c.created_at DESC
LIMIT 100;
```

------------------------------------------------------------------------

## 🧠 First observation: the JOINs were not the problem

I compared:

-   AS-IS version (3 JOINs on the same table)
-   TO-BE version (only 1 JOIN)

The result?

The execution plan showed in both cases:

``` text
Parallel Seq Scan on payment_report
Rows Removed by Filter: ~2,000,000
Buffers: shared read = hundreds of thousands
Execution Time: 14–18 seconds
```

Reducing the JOINs had only a marginal impact.

The real problem was something else.

------------------------------------------------------------------------

## 📌 The culprit: `LIKE '%value%'` without a proper index

A search with a leading wildcard (`%value%`) makes a normal B-Tree index
unusable.

PostgreSQL is forced to perform a sequential scan of the entire table.

In this specific case:

-   \~3 GB of data
-   hundreds of thousands of 8KB pages read
-   I/O bound workload
-   seconds of latency

This is not a matter of "bad SQL". It is an access path problem.

------------------------------------------------------------------------

## 🔬 Before creating an index: risk analysis

The client rightly asked:

> "If we create a trigram (GIN) index, do we risk slowing down payment
> transactions?"

This is where a frequently ignored concept comes into play: **churn**.

### What is churn?

It represents how much a table changes after rows are inserted.

High frequency of: - UPDATE - DELETE

→ high churn\
→ higher index maintenance cost\
→ possible write degradation

In our case:

Table `payment_report`: - \~12k inserts/day - 0 updates - 0 deletes - 0
dead tuples

Profile: **append-only**

This is the best possible scenario to introduce a GIN index.

------------------------------------------------------------------------

## 📊 Critical check: synchronous or batch?

The table did not contain an insertion timestamp.

Solution: indirect analysis.

I correlated rows in `payment_report` with the cart timestamp
(`payment_cart.created_at`) and analyzed hourly distribution.

Result:

-   continuous 24/7 pattern
-   daytime peaks
-   nighttime drop
-   perfect correlation with cart traffic

Conclusion: near real-time population, not nightly batch.

------------------------------------------------------------------------

## 🛠️ The solution

``` sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX CONCURRENTLY idx_payment_report_reference_trgm
ON reporting.payment_report
USING gin (reference_code gin_trgm_ops);
```

Precautions:

-   Create during an off-peak window
-   Use CONCURRENTLY mode
-   Monitor I/O during index build

------------------------------------------------------------------------

## 📈 Expected result

After the index:

-   Sequential scan eliminated
-   GIN index scan used
-   Drastic I/O reduction
-   Query time from seconds → milliseconds

------------------------------------------------------------------------

## 🎯 Key lesson

When a query is slow:

1.  Don't stop at the number of JOINs.
2.  Look at the execution plan.
3.  Identify whether the bottleneck is CPU or I/O.
4.  Evaluate churn before introducing a GIN index.
5.  Always measure before deciding.

Often the problem is not "optimizing the query".\
It is giving the planner the right index.

------------------------------------------------------------------------

## 💬 Why share this case?

Because this is an extremely common scenario:

-   Large tables
-   "Contains" search patterns
-   Fear of introducing GIN indexes
-   Concern about write performance degradation

With data in hand, the decision becomes technical, not emotional.

Optimization is not magic.\
It is measurement, plan analysis, and understanding real system
behavior.
