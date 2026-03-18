---
title: "AWR, ASH and the 10 minutes that saved a go-live"
description: "Friday evening, the night before a go-live. Performance collapses. Using AWR and ASH I found a hidden full table scan in a stored procedure in under ten minutes — and the production release went ahead."
date: "2026-02-10T10:00:00+01:00"
draft: false
translationKey: "oracle_awr_ash"
tags: ["awr", "ash", "performance", "tuning", "go-live", "diagnostic"]
categories: ["oracle"]
image: "oracle-awr-ash.cover.jpg"
---

Friday, 6:40 PM. I already had my jacket on, ready to leave. The phone buzzes. It's the project manager.

"Ivan, we have a problem. The system is crawling. The go-live is tomorrow morning."

It's not the first time I've received a call like that. But the tone was different. This wasn't the usual vague complaint about slowness. This was panic.

I reconnect to the VPN, open a session on the client's Oracle 19c database. First thing I do is a quick check:

``` sql
SELECT metric_name, value
FROM   v$sysmetric
WHERE  metric_name IN ('Database CPU Time Ratio',
                       'Database Wait Time Ratio',
                       'Average Active Sessions');
```

**CPU Time Ratio**: 12%. Under normal conditions it was above 80%.

**Average Active Sessions**: 47. On a server with 16 cores.

Forty-seven active sessions. The database was drowning.

------------------------------------------------------------------------

## 🔥 The symptoms

The development team had completed the last application code deploy that afternoon. Everything seemed to work on the test environment. But when they launched the pre-go-live verification batch — the one that simulates production load — response times exploded.

Queries that normally ran in 2-3 seconds were taking 45. Batches that finished in 20 minutes were still running after an hour. The dominant {{< glossary term="wait-event" >}}wait events{{< /glossary >}} were `db file sequential read` and `db file scattered read` — unmistakable signs of massive physical I/O.

Something was reading enormous amounts of data from disk. Something that wasn't there before.

------------------------------------------------------------------------

## 📊 AWR: the big picture

{{< glossary term="awr" >}}AWR{{< /glossary >}} — Automatic Workload Repository — is the most powerful diagnostic tool Oracle provides. Every hour, Oracle takes a {{< glossary term="snapshot-oracle" >}}snapshot{{< /glossary >}} of performance statistics and stores it in the internal repository. By comparing two snapshots, you get a report that tells you exactly what happened during that period.

I generated a manual snapshot to capture the current situation:

``` sql
EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;
```

Then I looked for available snapshots:

``` sql
SELECT snap_id, begin_interval_time, end_interval_time
FROM   dba_hist_snapshot
WHERE  begin_interval_time > SYSDATE - 1/6
ORDER BY snap_id DESC;
```

I had a snapshot from 6:00 PM (before the visible problem) and the one I had just created at 6:45 PM. I generated the AWR report:

``` sql
SELECT output
FROM   TABLE(DBMS_WORKLOAD_REPOSITORY.awr_report_text(
         l_dbid     => (SELECT dbid FROM v$database),
         l_inst_num => 1,
         l_bid      => 4523,   -- begin snapshot
         l_eid      => 4524    -- end snapshot
       ));
```

### What the report said

The **Top 5 Timed Foreground Events** section was telling:

| Event | Waits | Time (s) | % DB time |
|---|---|---|---|
| db file scattered read | 1,247,832 | 3,847 | 58.2% |
| db file sequential read | 423,109 | 1,205 | 18.2% |
| CPU + Wait for CPU | — | 892 | 13.5% |
| log file sync | 12,445 | 287 | 4.3% |
| direct path read | 8,221 | 198 | 3.0% |

`db file scattered read` at 58%. Those are {{< glossary term="full-table-scan" >}}full table scans{{< /glossary >}}. Something was reading entire tables, block by block, without using indexes.

The **SQL ordered by Elapsed Time** section showed a single SQL_ID consuming 71% of total database time: `g4f2h8k1nw3z9`.

Now I knew what to look for.

------------------------------------------------------------------------

## 🔍 ASH: the microscope

AWR had given me the big picture. But I needed to understand **when** that SQL started, **who** was running it, and **which program** had launched it.

{{< glossary term="ash" >}}ASH{{< /glossary >}} — Active Session History — records the state of every active session once per second. It is the DBA's microscope: where AWR shows you averages over an hour, ASH shows you what was happening second by second.

``` sql
SELECT sample_time,
       session_id,
       sql_id,
       sql_plan_hash_value,
       event,
       program,
       module
FROM   v$active_session_history
WHERE  sql_id = 'g4f2h8k1nw3z9'
  AND  sample_time > SYSDATE - 1/24
ORDER BY sample_time DESC;
```

The results were clear:

- **Program**: `JDBC Thin Client` — the Java batch application
- **Module**: `BatchVerificaProduzione`
- **Event**: `db file scattered read` in 92% of samples
- **First occurrence**: 5:12 PM — right after the afternoon deploy
- **SQL_PLAN_HASH_VALUE**: `2891047563`

The {{< glossary term="execution-plan" >}}execution plan{{< /glossary >}} had changed. Before the deploy, that query used a different plan.

------------------------------------------------------------------------

## 🧩 The execution plan

I retrieved the current plan:

``` sql
SELECT *
FROM   TABLE(DBMS_XPLAN.display_awr(
         sql_id          => 'g4f2h8k1nw3z9',
         plan_hash_value => 2891047563
       ));
```

The result made the problem immediately obvious:

```
---------------------------------------------------------------------------
| Id | Operation            | Name            | Rows  | Cost  |
---------------------------------------------------------------------------
|  0 | SELECT STATEMENT     |                 |       | 48721 |
|  1 |  HASH JOIN           |                 | 2.1M  | 48721 |
|  2 |   TABLE ACCESS FULL  | MOVIMENTI_TEMP  | 2.1M  | 41893 |
|  3 |   INDEX RANGE SCAN   | IDX_CLIENTI_PK  |     1 |     2 |
---------------------------------------------------------------------------
```

**TABLE ACCESS FULL on MOVIMENTI_TEMP**. A temporary table with 2.1 million rows, read in full every time. No index. No effective filter.

I checked what existed before the deploy by looking at the previous plan in AWR:

``` sql
SELECT plan_hash_value, timestamp
FROM   dba_hist_sql_plan
WHERE  sql_id = 'g4f2h8k1nw3z9'
ORDER BY timestamp;
```

The previous plan (hash `1384726091`) used an `INDEX RANGE SCAN` on an index that — as it turned out — **had been dropped during the deploy**. The migration script included a `DROP TABLE MOVIMENTI_TEMP` followed by a recreate, but without recreating the index.

------------------------------------------------------------------------

## ⚡ The fix

Ten minutes. From the moment I connected to the moment I identified the cause. Not because of skill — because of the tools.

The fix was straightforward:

``` sql
CREATE INDEX idx_movimenti_temp_cliente
ON movimenti_temp (id_cliente, data_movimento)
TABLESPACE idx_data;
```

After creating the index, I forced a re-parse of the query:

``` sql
EXEC DBMS_SHARED_POOL.purge('g4f2h8k1nw3z9', 'C');
```

I asked the team to relaunch the batch. Execution time: 18 minutes. Identical to previous tests.

The Saturday morning go-live proceeded as planned.

------------------------------------------------------------------------

## 📋 AWR vs ASH: when to use which

After that episode I formalised a rule I always follow:

| Characteristic | AWR | ASH |
|---|---|---|
| Granularity | Hourly snapshots (configurable) | Sample every second |
| Historical depth | Up to 30 days (default 8) | 1 hour in memory, then in AWR |
| Primary use case | Trend analysis, period comparison | Pinpoint diagnosis, SQL isolation |
| Primary view | `DBA_HIST_*` | `V$ACTIVE_SESSION_HISTORY` |
| Historical view | — | `DBA_HIST_ACTIVE_SESS_HISTORY` |
| Licence required | Diagnostic Pack | Diagnostic Pack |
| Typical output | HTML/text report | Ad hoc queries |

The rule of thumb: **AWR to understand what changed, ASH to understand why**.

AWR tells you: "Between 5:00 PM and 6:00 PM, 58% of database time was spent on full table scans." ASH tells you: "At 5:12:34 PM, session 847 was executing query g4f2h8k1nw3z9 with a full table scan on MOVIMENTI_TEMP, launched by the program BatchVerificaProduzione."

They are complementary. Using only one is like diagnosing a problem by looking only at the CT scan or only at the blood tests.

------------------------------------------------------------------------

## 🛡️ Queries every DBA should have ready

Over the years I've built a set of diagnostic queries that I always keep at hand. I share them because in an emergency there is no time to write them from scratch.

### Top SQL by execution time (last hour)

``` sql
SELECT sql_id,
       COUNT(*) AS samples,
       ROUND(COUNT(*) / 60, 1) AS est_minutes,
       MAX(event) AS top_event,
       MAX(program) AS program
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/24
  AND  sql_id IS NOT NULL
GROUP BY sql_id
ORDER BY samples DESC
FETCH FIRST 10 ROWS ONLY;
```

### Wait event distribution for a specific SQL

``` sql
SELECT event,
       COUNT(*) AS samples,
       ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER(), 1) AS pct
FROM   v$active_session_history
WHERE  sql_id = '&sql_id'
  AND  sample_time > SYSDATE - 1/24
GROUP BY event
ORDER BY samples DESC;
```

### Execution plan comparison over time

``` sql
SELECT plan_hash_value,
       MIN(timestamp) AS first_seen,
       MAX(timestamp) AS last_seen,
       COUNT(*) AS executions_in_awr
FROM   dba_hist_sqlstat
WHERE  sql_id = '&sql_id'
GROUP BY plan_hash_value
ORDER BY first_seen;
```

------------------------------------------------------------------------

## 🎯 What I learned that evening

Three lessons I carry with me.

**First**: a deploy is not just code. It is also structure. When you release to production, you must verify that indexes, constraints, statistics and grants are consistent with what was there before. A script that does `DROP TABLE` and `CREATE TABLE` without recreating the indexes is a time bomb.

**Second**: AWR and ASH are not tools for senior DBAs. They are front-line tools, like a defibrillator. You need to know how to use them before you need them, not during the emergency.

**Third**: ten minutes of correct diagnosis are worth more than three hours of blind attempts. When the system is on its knees, the temptation is to restart, kill sessions, throw more resources at it. But without knowing what is happening, you are shooting in the dark.

That evening I left the office at 7:20 PM. Forty minutes after the phone call. The next day the go-live went ahead without a hitch, and on Monday the system was running smoothly.

I'm not a hero. I just used the right tools.

------------------------------------------------------------------------

## Glossary

**[AWR](/en/glossary/awr/)** — Automatic Workload Repository. A built-in Oracle component that collects performance statistics through periodic snapshots and generates comparative diagnostic reports.

**[ASH](/en/glossary/ash/)** — Active Session History. An Oracle component that samples the state of every active session once per second, storing it in memory and then in AWR. It is the DBA's microscope for pinpoint diagnosis.

**[Full Table Scan](/en/glossary/full-table-scan/)** — A read operation where Oracle reads every block of a table without using indexes. In wait events it shows up as `db file scattered read`.

**[Wait Event](/en/glossary/wait-event/)** — A diagnostic event recorded by Oracle whenever a session cannot proceed because it is waiting for a resource (I/O, lock, CPU, network). Wait event analysis is the foundation of Oracle's diagnostic methodology.

**[Snapshot](/en/glossary/snapshot-oracle/)** — A point-in-time capture of performance statistics taken periodically by AWR (every 60 minutes by default). Comparing two snapshots generates the AWR report.
