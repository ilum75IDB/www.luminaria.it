---
title: "Wait Event"
description: "A diagnostic event recorded by Oracle whenever a session cannot proceed and must wait for a resource — I/O, lock, network or CPU."
translationKey: "glossary_wait_event"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Wait Event** is an Oracle Database diagnostic indicator that identifies why a session is waiting rather than actively working. Whenever a process cannot proceed — because it is waiting for a block from disk, a lock, a network response or a CPU slot — Oracle records a specific wait event.

## The most common

| Wait Event | Meaning |
|---|---|
| `db file sequential read` | Single-block read — typical of index access |
| `db file scattered read` | Multi-block read — typical of full table scans |
| `log file sync` | Waiting for commit to redo log |
| `enq: TX - row lock contention` | Row lock conflict |
| `direct path read` | Direct read (bypassing buffer cache) |

## What they're for

Wait events are the foundation of Oracle's diagnostic methodology. By analysing which events dominate DB time (via AWR or ASH) you can immediately identify the nature of the problem: I/O, contention, CPU or network.

## Where to find them

- **Real-time**: `V$SESSION_WAIT`, `V$ACTIVE_SESSION_HISTORY`
- **Historical**: AWR reports (Top Timed Foreground Events section), `DBA_HIST_ACTIVE_SESS_HISTORY`

The DBA's rule: don't guess what's slowing the database down — look at the wait events.
