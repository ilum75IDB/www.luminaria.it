---
title: "EXPLAIN ANALYZE is not enough: how to actually read a PostgreSQL execution plan"
description: "A real-world case where the optimizer chose a nested loop on 2 million rows because statistics were stale. How to read an execution plan, spot the red flags, and fix the root cause."
date: "2025-10-28T10:00:00+01:00"
draft: false
translationKey: "explain_analyze_postgresql"
tags: ["explain", "query-tuning", "execution-plan", "optimizer", "performance"]
categories: ["postgresql"]
image: "explain-analyze-postgresql.cover.jpg"
---

The other day a colleague sends me a screenshot on Teams. A query running on a 2-million-row table, 45 seconds execution time. He writes:

> "I ran EXPLAIN ANALYZE, but I can't figure out what's wrong. The plan looks fine."

Spoiler: the plan was anything but fine. The optimizer had chosen a {{< glossary term="nested-loop" >}}nested loop{{< /glossary >}} join where a {{< glossary term="hash-join" >}}hash join{{< /glossary >}} was needed, and the reason was trivial — stale statistics. But to get there I had to read the plan line by line, and that's when I realized that most DBAs I know use EXPLAIN ANALYZE as a binary oracle: if the time is high, the query is slow. End of analysis.

No. EXPLAIN ANALYZE is a diagnostic tool, not a verdict. You need to know how to read it.

------------------------------------------------------------------------

## 🔧 EXPLAIN, EXPLAIN ANALYZE, EXPLAIN (ANALYZE, BUFFERS): three different things

Let's start with the basics, because the confusion is more widespread than you'd think.

**EXPLAIN** alone shows the *estimated* plan. The optimizer decides what it would do, but doesn't execute anything. Useful for understanding the strategy, useless for understanding reality.

``` sql
EXPLAIN
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

**EXPLAIN ANALYZE** actually runs the query and adds real timings. Now you can see how long each node took, how many rows it actually returned. But there's a missing piece.

``` sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

**EXPLAIN (ANALYZE, BUFFERS)** is what I always use. It adds information about how many disk pages were read, how many were in cache (shared hit) and how many had to be loaded from disk (shared read). Without BUFFERS you're driving at night with no headlights.

``` sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

Personal rule: if someone sends me an EXPLAIN without BUFFERS, I send it back.

------------------------------------------------------------------------

## 📖 Anatomy of a node: what to read and in what order

An {{< glossary term="execution-plan" >}}execution plan{{< /glossary >}} is a tree. Each node looks like this:

``` text
->  Hash Join  (cost=1234.56..5678.90 rows=50000 width=120)
      (actual time=12.345..89.012 rows=48750 loops=1)
      Buffers: shared hit=1200 read=3400
```

Here's what to look at:

**cost** — two numbers separated by `..`. The first is the startup cost (how much before returning the first row), the second is the total estimated cost. These are arbitrary optimizer units, not milliseconds. They're useful for comparing alternative plans, not for measuring absolute performance.

**rows** — the rows estimated by the optimizer. Compare them with `actual rows`. If there's an order of magnitude difference, you've found the problem.

**actual time** — real time in milliseconds. Again two values: startup and total. Watch the `loops` field: if loops=10, the total time should be multiplied by 10.

**Buffers** — `shared hit` are pages found in memory, `shared read` are pages read from disk. If `read` dominates, your working set doesn't fit in RAM.

------------------------------------------------------------------------

## 🚨 The number one red flag: estimated rows vs actual rows

Back to my colleague's case. The plan showed:

``` text
->  Nested Loop  (cost=0.87..45678.12 rows=150 width=200)
      (actual time=0.034..44890.123 rows=1950000 loops=1)
```

The optimizer estimated 150 rows. In reality, almost 2 million arrived.

When the estimate is off by 4 orders of magnitude, the plan is inevitably wrong. The optimizer chose a nested loop because it thought it was iterating over 150 rows. A nested loop on 150 rows is lightning fast. On 2 million, it's a disaster.

A hash join or merge join would have been the right choice. But the optimizer couldn't know that with the statistics it had.

Rule of thumb: if the ratio between estimated and actual rows exceeds 10x, you have a statistics problem. Above 100x, the plan is almost certainly wrong.

------------------------------------------------------------------------

## 🔍 Why statistics lie

PostgreSQL maintains table statistics in `pg_statistic` (readable through `pg_stats`). These statistics include:

- value distribution (most common values)
- value histogram
- number of distinct values
- NULL percentage

The optimizer uses this information to estimate the selectivity of every WHERE condition and the cardinality of every join.

The problem? Statistics are updated by {{< glossary term="postgresql-analyze" >}}`ANALYZE`{{< /glossary >}} — which can be manual or handled by autovacuum. But autovacuum triggers ANALYZE only when the number of modified rows exceeds a threshold:

``` text
threshold = autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × n_live_tuples
```

Defaults: 50 rows + 10% of live tuples. On a 2-million-row table, that means 200,000 modifications before an automatic ANALYZE kicks in.

In my colleague's case, the `orders` table had grown from 500,000 to 2 million rows in three weeks — a massive import from a legacy system. Autovacuum hadn't refreshed the statistics because 10% of 500,000 (the known size) was 50,000, and the rows had been inserted in batches that individually never crossed the threshold.

Result: the optimizer was still reasoning as if the table had 500,000 rows with the old value distribution.

------------------------------------------------------------------------

## 🛠️ Updating statistics: the first thing to do

The immediate solution was obvious:

``` sql
ANALYZE orders;
```

After the ANALYZE, I re-ran the query with EXPLAIN (ANALYZE, BUFFERS):

``` text
->  Hash Join  (cost=8500.00..32000.00 rows=1940000 width=200)
      (actual time=120.000..2800.000 rows=1950000 loops=1)
      Buffers: shared hit=28000 read=4500
```

From 45 seconds to under 3 seconds. The optimizer had chosen a hash join, the row estimate was accurate, and the plan was completely different.

But I didn't stop there. If the problem happened once, it will happen again.

------------------------------------------------------------------------

## 📊 default_statistics_target: when 100 is not enough

PostgreSQL collects 100 sample values per column by default. For small tables or uniform distributions, that's fine. For large tables with skewed distributions, 100 samples can give a distorted picture.

In the `orders` table case, the `customer_id` column had a very skewed distribution: 5% of customers generated 60% of orders. With 100 samples, the optimizer couldn't capture this asymmetry.

The solution:

``` sql
ALTER TABLE orders
ALTER COLUMN customer_id SET STATISTICS 500;

ANALYZE orders;
```

After raising the {{< glossary term="postgresql-default-statistics-target" >}}target{{< /glossary >}} to 500, the optimizer's cardinality estimates for joins with `customers` became much more accurate.

Rule: if a column is frequently used in WHERE or JOIN clauses and has non-uniform distribution, raise the target. 500 is a good starting point. You can go up to 1000, but beyond that it rarely helps and slows down ANALYZE itself.

------------------------------------------------------------------------

## ⚠️ When to force the planner: enable_nestloop and enable_hashjoin

Sometimes, even with fresh statistics, the optimizer takes the wrong path. It happens with complex queries, many joined tables, or when column correlations mislead the estimates.

PostgreSQL offers parameters to disable specific strategies:

``` sql
SET enable_nestloop = off;
```

This forces the optimizer not to use nested loops. It's not a solution, it's a diagnostic band-aid. If you disable nested loops and the query drops from 45 seconds to 3 seconds, you've confirmed the join strategy was the problem. But you can't leave `enable_nestloop = off` in production because there are a thousand queries where nested loops are the right choice.

I use these parameters in only two scenarios:

1. **Diagnostics**: to confirm which join strategy is the problem
2. **Emergency**: when the business is down and you need to get a critical query running while you look for the real fix

After diagnostics, the correct fix is always on statistics, indexes, or query rewriting.

------------------------------------------------------------------------

## 📋 My workflow when a query is slow

After thirty years doing this job, my process has become almost mechanical:

**1. EXPLAIN (ANALYZE, BUFFERS)** — always with BUFFERS. I save the complete output, not just the last few lines.

**2. Look for row discrepancies** — I compare estimated `rows=` with actual `rows=` on every node. I start from the leaf nodes and work up to the root. The first significant discrepancy is almost always the cause.

**3. Check the statistics** — I look at `pg_stats` for the involved columns. I verify `last_autoanalyze` and `last_analyze` in `pg_stat_user_tables`. If the last ANALYZE is old, I run it and re-evaluate.

**4. Evaluate BUFFERS** — if `shared read` is very high compared to `shared hit`, the problem might be I/O, not the plan. In that case the fix is `shared_buffers` or the working set simply doesn't fit in RAM.

**5. Test alternatives** — if statistics are fresh but the plan is still wrong, I use `enable_nestloop`, `enable_hashjoin`, `enable_mergejoin` to understand which strategy works best. Then I try to guide the optimizer toward that strategy with indexes or query rewriting.

Nothing spectacular. No magic tricks. Just systematic reading of the plan, one line at a time.

------------------------------------------------------------------------

## 💬 The lesson from that day

My colleague, after seeing the difference, told me: "So all it took was an ANALYZE?"

Yes and no. In that specific case, yes. But the point isn't the command. The point is knowing how to read the plan to understand *where* to look. EXPLAIN ANALYZE gives you the data. It's up to you to interpret it.

I've seen DBAs with years of experience run EXPLAIN ANALYZE, look at the total time at the bottom, and say "the query is slow." It's like checking a patient's temperature and saying "they have a fever." Sure, but what's causing it?

The execution plan tells you what's causing it. Each node is an organ. Estimated rows versus actual rows are the lab results. Buffers are the X-rays. And ANALYZE is the antibiotic that solves 70% of cases.

But for that remaining 30%, you need to read. Line by line. Node by node. There's no shortcut.

------------------------------------------------------------------------

## Glossary

**[Execution Plan](/en/glossary/execution-plan/)** — the sequence of operations (scan, join, sort) the database chooses to resolve a SQL query. Viewed with EXPLAIN and EXPLAIN ANALYZE.

**[Nested Loop](/en/glossary/nested-loop/)** — a join strategy that for each row in the outer table looks for matches in the inner table. Ideal for few rows, disastrous on large volumes when mistakenly chosen by the optimizer.

**[Hash Join](/en/glossary/hash-join/)** — a join strategy that builds a hash table from the smaller table and then scans the larger one looking for matches with O(1) lookups. Efficient on large volumes without indexes.

**[ANALYZE](/en/glossary/postgresql-analyze/)** — PostgreSQL command that collects statistics on data distribution in tables, used by the optimizer to estimate cardinality and choose the execution plan.

**[default_statistics_target](/en/glossary/postgresql-default-statistics-target/)** — PostgreSQL parameter that defines how many samples to collect per column during ANALYZE. The default is 100; for columns with skewed distribution it should be raised to 500-1000.
