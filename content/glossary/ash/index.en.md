---
title: "ASH"
description: "Active Session History — Oracle component that records the state of every active session once per second, used for pinpoint performance diagnosis."
translationKey: "glossary_ash"
aka: "Active Session History"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**ASH** (Active Session History) is an Oracle Database component that samples the state of every active session once per second and stores the data in an in-memory circular buffer (the `V$ACTIVE_SESSION_HISTORY` view).

## How it works

Every second Oracle records for each active session:

- Currently executing SQL (`SQL_ID`)
- Current wait event
- Calling program and module
- Execution plan in use (`SQL_PLAN_HASH_VALUE`)

Older data is automatically flushed to AWR tables (`DBA_HIST_ACTIVE_SESS_HISTORY`) and retained for the configured period.

## What it's for

ASH is the DBA's microscope: where AWR shows averages over hourly intervals, ASH lets you reconstruct what a single session was doing at a precise moment. It is the ideal tool for:

- Identifying who is running a problematic SQL
- Understanding exactly when a problem started (to the second)
- Correlating sessions, programs and wait events in real time

## When to use it

Use it when the AWR report has already identified a dominant SQL or wait event and you need detail: which session, which program, at what exact time. The rule of thumb: **AWR to understand what changed, ASH to understand why**.
