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

A search with a leading wildcard (`%value%`) makes a normal {{< glossary term="b-tree" >}}B-Tree{{< /glossary >}} index
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

This is where a frequently ignored concept comes into play: {{< glossary term="churn" >}}**churn**{{< /glossary >}}.

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
CREATE EXTENSION IF NOT EXISTS {{< glossary term="pg-trgm" >}}pg_trgm{{< /glossary >}};

CREATE INDEX CONCURRENTLY idx_payment_report_reference_trgm
ON reporting.payment_report
USING {{< glossary term="gin-index" >}}gin{{< /glossary >}} (reference_code gin_trgm_ops);
```

Precautions:

-   Create during an off-peak window
-   Use CONCURRENTLY mode
-   Monitor I/O during index build

------------------------------------------------------------------------

## 📈 Result: the execution plan before and after

Here is the full execution plan for the query — before and after creating the trigram index.

**Before** (without trigram index):

``` text
Nested Loop Inner Join
  → Nested Loop Inner Join
    → Nested Loop Inner Join
      → Seq Scan on payment_report as r
          Filter: ((reference_code)::text ~~ '%ABC123%'::text)
      → Index Scan using payment_cart_pkey on payment_cart as c
          Filter: (service_id = 1001)
          Index Cond: (id = r.cart_id)
    → Index Only Scan using payment_cart_pkey on payment_cart as c2
        Index Cond: (id = c.id)
  → Index Only Scan using payment_cart_pkey on payment_cart as c3
      Index Cond: (id = c.id)
```

**After** (with trigram index):

``` text
Nested Loop Inner Join
  → Nested Loop Inner Join
    → Nested Loop Inner Join
      → Bitmap Heap Scan on payment_report as r
          Recheck Cond: ((reference_code)::text ~~ '%ABC123%'::text)
        → Bitmap Index Scan using idx_payment_report_reference_trgm
            Index Cond: ((reference_code)::text ~~ '%ABC123%'::text)
      → Index Scan using payment_cart_pkey on payment_cart as c
          Filter: (service_id = 1001)
          Index Cond: (id = r.cart_id)
    → Index Only Scan using payment_cart_pkey on payment_cart as c2
        Index Cond: (id = c.id)
  → Index Only Scan using payment_cart_pkey on payment_cart as c3
      Index Cond: (id = c.id)
```

The key change is at steps 4–5: the `Seq Scan` — which read the entire table row by row — has been replaced by a `Bitmap Heap Scan` driven by the trigram index `idx_payment_report_reference_trgm`. PostgreSQL now filters directly through the index and only rechecks the candidate rows.

Same query, same data, but a completely different access path. From seconds to milliseconds.

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

------------------------------------------------------------------------

## Glossary

**[GIN Index](/en/glossary/gin-index/)** — Generalized Inverted Index: PostgreSQL index type that creates an inverted mapping from each element to the records containing it. Ideal for "contains" searches on text with pg_trgm.

**[B-Tree](/en/glossary/b-tree/)** — Balanced tree data structure, the default index in relational databases. Efficient for equality and range searches, but unusable for `LIKE '%value%'`.

**[pg_trgm](/en/glossary/pg-trgm/)** — PostgreSQL extension that decomposes text into trigrams (3-character sequences), enabling GIN indexes to accelerate wildcard searches.

**[Churn](/en/glossary/churn/)** — Measure of how much a table changes after insertion. Low churn (append-only) is the best scenario for introducing a GIN index without degrading writes.

**[Execution Plan](/en/glossary/execution-plan/)** — Sequence of operations chosen by the database to resolve a query. Reading the plan is the first step to identify whether the problem is CPU, I/O or a wrong access path.
