---
title: "VACUUM and autovacuum: why PostgreSQL needs someone to clean up"
description: "A 200 GB PostgreSQL database with tables bloated to three times their actual size. Autovacuum was enabled but poorly configured. How to diagnose bloat, read pg_stat_user_tables, and tune without disabling anything."
date: "2026-03-24T08:03:00+01:00"
lastmod: "2026-03-24T08:03:00+01:00"
draft: false
translationKey: "vacuum_autovacuum_postgresql"
tags: ["vacuum", "autovacuum", "mvcc", "performance", "bloat"]
categories: ["postgresql"]
image: "vacuum-autovacuum-postgresql.cover.jpg"
---

A couple of years ago I was asked to look at a production PostgreSQL
instance that "slows down every week". Always the same pattern: Monday
is fine, Friday is a disaster. Someone restarts the service over the
weekend and the cycle starts again.

Database around 200 GB. Main tables occupying nearly three times their
actual data size. Queries falling into sequential scans where they
shouldn't have been. Response times climbing day after day.

Autovacuum was enabled. Nobody had disabled it. But nobody had
configured it either.

------------------------------------------------------------------------

## 🧠 MVCC: why PostgreSQL generates "garbage"

To understand the problem, you need a step back. PostgreSQL uses MVCC —
Multi-Version Concurrency Control. Every time you run an UPDATE, the
database doesn't overwrite the original row. It creates a new version
and marks the old one as "dead".

Same for DELETEs: the row isn't physically removed. It's marked as no
longer visible to new transactions.

These dead rows are called **dead tuples**. They stay inside the data
pages, taking up disk space and slowing down scans.

That's the price PostgreSQL pays for transactional isolation without
exclusive read locks. A fair price — as long as someone sweeps up
afterwards.

------------------------------------------------------------------------

## 🔧 VACUUM: what it actually does

The `VACUUM` command does one simple thing: it reclaims space taken by
dead tuples and makes it reusable for new inserts.

It doesn't return space to the operating system. It doesn't reorganize
the table. It doesn't compact anything. It marks pages as rewritable.

``` sql
VACUUM reporting.transactions;
```

That's enough in most cases. VACUUM is lightweight, doesn't block
writes, and can run alongside normal queries.

### What about `VACUUM FULL`?

`VACUUM FULL` is a different beast. It physically rewrites the entire
table, eliminating all dead space. It returns space to the filesystem.

But the cost is brutal: **it takes an exclusive lock** on the table for
the entire duration. No reads, no writes. On large tables we're talking
minutes or hours.

``` sql
VACUUM FULL reporting.transactions;
```

In production, `VACUUM FULL` should be used very rarely. In emergencies.
And always off-hours.

------------------------------------------------------------------------

## ⚙️ Autovacuum: the silent janitor

PostgreSQL has a daemon that runs VACUUM automatically: autovacuum.

It kicks in when a table accumulates enough dead tuples. The threshold
is calculated like this:

```
vacuum threshold = autovacuum_vacuum_threshold
                 + autovacuum_vacuum_scale_factor × n_live_tup
```

The defaults:

- `autovacuum_vacuum_threshold`: **50** dead tuples
- `autovacuum_vacuum_scale_factor`: **0.2** (20%)

In plain terms: on a table with 10 million rows, autovacuum fires when
dead tuples exceed **2,000,050**. Two million dead rows before anyone
cleans up.

For a table with 500,000 updates per day, that means autovacuum
triggers maybe every 4 days. In the meantime bloat grows, scans slow
down, indexes swell.

That's why Monday was fine and Friday was a disaster.

------------------------------------------------------------------------

## 📊 Diagnostics: reading pg_stat_user_tables

The first thing to do when you suspect a vacuum problem is to query
`pg_stat_user_tables`:

``` sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    autovacuum_count,
    vacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

In my client's case, the picture looked like this:

``` text
relname            | n_live_tup | n_dead_tup | dead_pct | last_autovacuum
-------------------+------------+------------+----------+------------------
transactions       | 12,400,000 |  3,800,000 |   23.5%  | 3 days ago
order_lines        |  8,200,000 |  2,100,000 |   20.4%  | 4 days ago
inventory_moves    |  5,600,000 |  1,900,000 |   25.3%  | 5 days ago
```

Nearly a quarter of the rows were dead. Autovacuum was running, but far
too infrequently to keep up.

------------------------------------------------------------------------

## 🎯 Tuning: adapting autovacuum to reality

The trick isn't to disable autovacuum. Never. The trick is to configure
it for the tables that need it.

PostgreSQL lets you set autovacuum parameters **per table**:

``` sql
ALTER TABLE reporting.transactions SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000
);
```

With this setting, autovacuum fires after 1,000 + 1% of live rows worth
of dead tuples. On 12 million rows, it kicks in at ~121,000 dead tuples
instead of 2 million.

### cost_delay: don't throttle the vacuum

Another critical parameter is `autovacuum_vacuum_cost_delay`. It
controls how much vacuum "slows itself down" to avoid overloading I/O.

The default is 2 milliseconds. On modern servers with SSDs, that's too
conservative. Reducing it to 0 or 1 ms lets vacuum finish faster:

``` sql
ALTER TABLE reporting.transactions SET (
    autovacuum_vacuum_cost_delay = 0
);
```

### max_workers

The default is 3 autovacuum workers. If you have dozens of
high-traffic tables, 3 workers aren't enough. Consider raising to 5–6,
while monitoring CPU and I/O impact:

``` text
-- in postgresql.conf
autovacuum_max_workers = 5
```

------------------------------------------------------------------------

## 📏 Measuring bloat

How do you know how much space your tables are wasting?

The classic query uses `pgstattuple`:

``` sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    pg_size_pretty(pg_total_relation_size('reporting.transactions')) AS total_size,
    pg_size_pretty(pg_total_relation_size('reporting.transactions')
                   - pg_relation_size('reporting.transactions')) AS index_size,
    *
FROM pgstattuple('reporting.transactions');
```

Key fields: `dead_tuple_percent` and `free_space`. If dead_tuple
exceeds 20–30%, the table has a serious problem.

A less precise but lighter alternative is estimating bloat ratio by
comparing `pg_class.relpages` with estimated rows — there are
well-known queries in the community for this (the classic "bloat
estimation query" from PostgreSQL Experts).

------------------------------------------------------------------------

## 🛠️ When VACUUM isn't enough: pg_repack

If bloat is already out of control — tables at 50–70% dead space —
regular VACUUM won't reclaim everything. It frees dead tuples, but
fragmented space remains.

`VACUUM FULL` works but locks everything.

The production alternative is **pg_repack**: it rebuilds the table
online, without prolonged exclusive locks.

``` bash
pg_repack -d mydb -t reporting.transactions
```

This isn't a weekly solution. It's the heavy-duty fix for when things
have already gone south. The real solution is to never get there, with
a well-configured autovacuum.

------------------------------------------------------------------------

## 💬 The principle

Disabling autovacuum is the worst thing you can do to a production
PostgreSQL. I've seen it done "because it slows down queries during the
day". Sure, because in the meantime bloat is eating your database from
the inside.

Autovacuum with PostgreSQL defaults is designed for a generic database.
No production database is generic. Every table has its own write pattern,
its own volume, its own rhythm.

Three things to take away:

1. Check `pg_stat_user_tables` regularly. If `n_dead_tup` grows faster
   than autovacuum can clean, you have a problem.

2. Configure `scale_factor` and `threshold` for high-traffic tables.
   There's no universal configuration.

3. Don't wait until bloat reaches 50% to act. At that point your
   options are few and all painful.

Databases don't maintain themselves. Not even the ones that have a
daemon trying to.

------------------------------------------------------------------------

## Glossary

**[VACUUM](/en/glossary/vacuum/)** — PostgreSQL command that reclaims space occupied by dead tuples, making it reusable for new inserts without returning it to the operating system.

**[MVCC](/en/glossary/mvcc/)** — Multi-Version Concurrency Control — PostgreSQL's concurrency model that maintains multiple row versions to ensure transactional isolation without exclusive locks on reads.

**[Dead Tuple](/en/glossary/dead-tuple/)** — Obsolete row in a PostgreSQL table, marked as no longer visible after an UPDATE or DELETE but not yet physically removed from disk.

**[Autovacuum](/en/glossary/autovacuum/)** — PostgreSQL daemon that automatically runs VACUUM and ANALYZE on tables when the number of dead tuples exceeds a configurable threshold.

**[Bloat](/en/glossary/bloat/)** — Dead space accumulated in a PostgreSQL table or index due to unremoved dead tuples, inflating disk size and degrading query performance.
