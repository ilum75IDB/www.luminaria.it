---
title: "From Single Instance to Data Guard: The Day the CEO Understood DR"
description: "An Oracle production database with no redundancy. A disk failure that brought everything to a halt for six hours. And the CEO's decision to invest in an Active Data Guard architecture with automatic switchover."
date: "2025-12-16T10:00:00+01:00"
draft: false
translationKey: "oracle_data_guard"
tags: ["data-guard", "disaster-recovery", "high-availability", "switchover", "architecture"]
categories: ["oracle"]
image: "oracle-data-guard.cover.jpg"
---

The client was a mid-sized insurance company. Three hundred employees, an in-house management application running on Oracle 19c, a single physical server in the ground-floor server room. No replica. No standby. No disaster recovery plan.

For five years everything had worked. And when things work, nobody wants to spend money protecting against problems they've never seen.

## The day everything stopped

On a Wednesday morning in November, at 8:47 AM, the primary data group's disk suffered a hardware failure. Not a logical error, not a recoverable corruption. A physical failure. The RAID controller lost two disks simultaneously — one had been degraded for weeks without anyone noticing, the other gave out suddenly.

The database stopped. Policies couldn't be issued. Claims couldn't be processed. The call center told customers "technical problems, please call back later."

I got the call at 9:15. When I arrived on site, the sysadmin was already looking for compatible disks. He found them in the early afternoon. Between disk replacement, RAID rebuild, and database recovery from the previous night's backup, the system was back online at 3:20 PM.

Six and a half hours of total downtime. And the loss of all transactions from 11:00 PM the night before to 8:47 AM — roughly ten hours of data, because the backup was nightly only and archived logs weren't being copied to another machine.

That evening the CEO sent an email to the entire company. The next day he called me: "What do we need to do so this never happens again?"

## The design

The answer was simple in concept, less so in execution: they needed a second database, synchronized in real time, ready to take over if the primary failed.

Oracle Active {{< glossary term="data-guard" >}}Data Guard{{< /glossary >}} does exactly this. A primary database generates {{< glossary term="redo-log" >}}redo logs{{< /glossary >}}, and a standby receives and continuously applies them. If the primary dies, the standby becomes primary. If everything is fine, the standby can also be used in read-only mode — for reports, for backups, to offload the primary.

I designed a two-node architecture:

- **Primary** (`oraprod1`): the existing server, with new disks, at headquarters
- **Standby** (`oraprod2`): a new identical server, at the hosting provider's data center, 12 km away

The distance wasn't random. Far enough to survive a localized event (fire, flood, prolonged power outage), close enough to allow synchronous replication without noticeable latency.

## The configuration

### Preparing the primary

The first step was verifying that the primary was in `ARCHIVELOG` mode with `FORCE LOGGING` enabled. Without these two prerequisites, Data Guard has nothing to replicate.

```sql
-- Check archivelog mode
SELECT log_mode FROM v$database;

-- If needed, enable it
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Force logging: prevents NOLOGGING operations
ALTER DATABASE FORCE LOGGING;
```

`FORCE LOGGING` is critical. Without it, any operation with a `NOLOGGING` clause — a `CREATE TABLE AS SELECT`, an `ALTER INDEX REBUILD` — won't generate redo and creates gaps in replication. I've seen it happen three times in my career. After the third time, I decided `FORCE LOGGING` is always on, no exceptions.

### Standby redo logs

On the primary, I created standby redo logs — dedicated groups that will be used when (and if) this server becomes the standby after a switchover.

```sql
-- Standby redo logs: n+1 relative to online redo logs
-- If you have 3 online groups, create 4 standby groups
ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 SIZE 200M;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 SIZE 200M;
```

The rule is n+1: if the primary has three redo log groups, the standby needs four. It's not documented very clearly, but I learned it the hard way — with three equal groups, under heavy load the standby can stall waiting for a free group.

### Network configuration

The `tnsnames.ora` on both nodes needs to know about both the primary and the standby. The configuration is symmetrical:

```
ORAPROD1 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.10.1)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = oraprod)
    )
  )

ORAPROD2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.0.5.1)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = oraprod)
    )
  )
```

The `listener.ora` on the standby must include a static entry for the database, because during the restore the standby isn't open yet and the listener can't register it dynamically:

```
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = oraprod_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/19c)
      (SID_NAME = oraprod)
    )
  )
```

The `_DGMGRL` suffix is used by the Data Guard Broker to identify the instance. Without this static entry, the broker can't connect to the standby and switchover operations fail with cryptic errors that cost you half a day.

### Creating the standby

For the initial database copy, I used an {{< glossary term="rman" >}}RMAN{{< /glossary >}} `DUPLICATE` over the network. No tape backup, no manual file transfers. Direct, from primary to standby:

```
-- On the standby server, start the instance in NOMOUNT
STARTUP NOMOUNT PFILE='/u01/app/oracle/product/19c/dbs/initoraprod.ora';

-- From RMAN, connected to both
RMAN TARGET sys/password@ORAPROD1 AUXILIARY sys/password@ORAPROD2

DUPLICATE TARGET DATABASE
  FOR STANDBY
  FROM ACTIVE DATABASE
  DORECOVER
  SPFILE
    SET db_unique_name='oraprod_stby'
    SET log_archive_dest_2=''
    SET fal_server='ORAPROD1'
  NOFILENAMECHECK;
```

`NOFILENAMECHECK` is used when file paths are identical on both machines — same directory structure, same naming convention. If paths differ, you need `DB_FILE_NAME_CONVERT` and `LOG_FILE_NAME_CONVERT` parameters.

The copy took about three hours for 400 GB over a dedicated 1 Gbps line. Not the fastest, but it's a one-time operation.

### Data Guard Broker

The Broker is the component that manages the Data Guard configuration centrally and allows switchover with a single command. Without the Broker you can do everything manually, but you don't want to do it manually when the primary just went down and the CEO is calling every five minutes.

```sql
-- On the primary
ALTER SYSTEM SET dg_broker_start=TRUE;

-- On the standby
ALTER SYSTEM SET dg_broker_start=TRUE;
```

Then, from `DGMGRL` on the primary:

```
DGMGRL> CREATE CONFIGURATION dg_config AS
         PRIMARY DATABASE IS oraprod
         CONNECT IDENTIFIER IS ORAPROD1;

DGMGRL> ADD DATABASE oraprod_stby AS
         CONNECT IDENTIFIER IS ORAPROD2
         MAINTAINED AS PHYSICAL;

DGMGRL> ENABLE CONFIGURATION;
```

At that point, `SHOW CONFIGURATION` should return:

```
Configuration - dg_config

  Protection Mode: MaxPerformance
  Members:
    oraprod   - Primary database
    oraprod_stby - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS
```

The word you want to see is `SUCCESS`. Anything else means there's a network, configuration, or permissions issue to resolve before moving forward.

## The first switchover

Two weeks after the architecture went live, I ran the first switchover test. On a Saturday morning, with the application shut down, but with the CEO present — he wanted to see it with his own eyes.

```
DGMGRL> SWITCHOVER TO oraprod_stby;
```

One command. Forty-two seconds. The primary became the standby, the standby became the primary. The applications, configured with the correct service, reconnected automatically.

```
DGMGRL> SHOW CONFIGURATION;

Configuration - dg_config

  Protection Mode: MaxPerformance
  Members:
    oraprod_stby - Primary database
    oraprod      - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS
```

Then we did the switchback — returning to the original primary. Another thirty-eight seconds. Clean.

The CEO looked at the screen, looked at me, and said: "Forty-two seconds versus six hours. Why didn't we do this before?"

I didn't give him the answer. We both knew it.

## What they don't tell you

The configuration I've described works. But there are things that Oracle's documentation doesn't emphasize enough.

**The network gap.** Synchronous replication (`SYNC`) guarantees zero data loss but introduces latency on every commit. With 12 km and a good fiber link, the added latency was 1-2 milliseconds — acceptable. But at 100 km it would have been 5-8 ms, and on an application with thousands of commits per second, the slowdown would be noticeable. That's why I chose `MaxPerformance` mode (asynchronous) as the default, accepting the theoretical possibility of losing a few seconds of transactions in case of a total disaster. For that client, losing five seconds of data was infinitely better than losing ten hours.

**The password file.** The `SYS` user's password file must be identical on both primary and standby. If you change it on one and not the other, redo transport stops silently. No obvious error, just a growing gap. I discovered this after an hour of debugging on a Sunday evening.

**Temp tablespaces.** The standby doesn't replicate temporary tablespaces. If you open the standby in read-only mode for reports (Active Data Guard), you need to manually create temp tablespaces, otherwise queries with sorts or hash joins fail with errors that have nothing to do with the real problem.

```sql
-- On the standby opened in read-only mode
ALTER TABLESPACE TEMP ADD TEMPFILE SIZE 2G AUTOEXTEND ON;
```

**Patches.** Primary and standby must be at the same patch level. If you apply a PSU to the primary without applying it to the standby, the redo might contain structures that the standby can't interpret. The switchover will work, but afterward you might have silent corruption. The correct procedure is: patch the standby first, switchover, patch the old primary (now standby), switchback.

## The numbers

Six months after implementation, the results were clear:

| Metric | Before | After |
|--------|--------|-------|
| {{< glossary term="rpo" >}}RPO{{< /glossary >}} (Recovery Point Objective) | ~10 hours (nightly backup) | < 5 seconds |
| {{< glossary term="rto" >}}RTO{{< /glossary >}} (Recovery Time Objective) | 6+ hours (restore from backup) | < 1 minute (switchover) |
| Parallel report availability | No | Yes (Active Data Guard) |
| Additional infrastructure cost | — | 1 server + dedicated line |
| Switchover tests performed | 0 | 6 (one per month) |

The total project cost — server, licenses, dedicated line, implementation — was roughly a quarter of what that single day of downtime had cost. Not in technical terms. In terms of policies not issued, claims not processed, customers not served.

## What I learned

Disaster recovery isn't a technical problem. It's a risk perception problem. As long as the database is running, DR is an expense. When the database stops, DR is an investment that should have been made six months earlier.

You can't convince a CEO with an architectural diagram. You can only wait for the disaster to happen and then be ready with the solution. It's cynical, but that's how it works in ninety percent of cases.

The only thing you can do beforehand is document the risk, put it in writing that you flagged it, and keep the project ready in the drawer. I had proposed that project eighteen months earlier. It had been shelved with a "let's revisit it next year."

Next year arrived on a Wednesday morning in November, at 8:47 AM.

------------------------------------------------------------------------

## Glossary

**[Data Guard](/en/glossary/data-guard/)** — Oracle technology for real-time database replication to one or more standby servers. The standby continuously receives and applies the primary's redo logs, enabling switchover in seconds.

**[Redo Log](/en/glossary/redo-log/)** — Log files where Oracle records every data change before writing it to the datafiles. They are the foundation of recovery and Data Guard replication: without redo, none of these operations is possible.

**[RPO](/en/glossary/rpo/)** — Recovery Point Objective. The maximum amount of data an organisation can afford to lose in a disaster, measured in time. With asynchronous Data Guard it is reduced to a few seconds.

**[RTO](/en/glossary/rto/)** — Recovery Time Objective. The maximum acceptable time to restore service after a failure. With Data Guard and automatic switchover, it goes from hours to under a minute.

**[RMAN](/en/glossary/rman/)** — Recovery Manager. Oracle's native tool for backup, restore and recovery, including standby database creation via `DUPLICATE ... FOR STANDBY FROM ACTIVE DATABASE`.
