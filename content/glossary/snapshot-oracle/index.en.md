---
title: "Snapshot (Oracle)"
description: "A point-in-time capture of performance statistics taken periodically by AWR and used to generate comparative diagnostic reports."
translationKey: "glossary_snapshot_oracle"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Snapshot** in Oracle is a point-in-time capture of database performance statistics stored in the AWR repository. By default Oracle generates a snapshot every 60 minutes and retains them for 8 days.

## How it works

Each snapshot records hundreds of metrics: wait events, SQL statistics, memory metrics (SGA, PGA), I/O by datafile, system statistics. Comparing two snapshots generates the AWR report, which shows what changed between the two points in time.

## Manual snapshots

In emergency situations you can generate a manual snapshot to capture the current state:

    EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;

This is useful when you want an immediate reference point — for example, before and after a deploy — without waiting for the automatic cycle.

## Management

Snapshots are accessible through the `DBA_HIST_SNAPSHOT` view. Retention (how many days to keep them) and interval (how often to generate them) are configured with:

    EXEC DBMS_WORKLOAD_REPOSITORY.modify_snapshot_settings(
      retention => 43200,   -- 30 days in minutes
      interval  => 30       -- every 30 minutes
    );

## Why they matter

Without snapshots, there is no AWR. Without AWR, performance diagnosis becomes guesswork instead of data-driven analysis. Snapshots are the foundation of observability in Oracle.
