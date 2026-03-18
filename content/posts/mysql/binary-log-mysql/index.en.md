---
title: "Binary logs in MySQL: what they are, how to manage them, and when you can delete them"
description: "A MySQL server with disk at 95%, 180 GB of binary logs accumulated over six months. From there, a deep dive into binlogs: what they contain, why they exist, how they work with replication and point-in-time recovery, and above all how to manage them without breaking things."
date: "2026-03-31T10:00:00+01:00"
draft: false
translationKey: "binary_log_mysql"
tags: ["binlog", "replication", "disk-space", "recovery", "mariadb"]
categories: ["mysql"]
image: "binary-log-mysql.cover.jpg"
---

The message in the infrastructure team's Slack channel was the kind that makes you look up from your screen: "Disk at 95% on the production db. Anyone able to look?"

The server was a MySQL 8.0 on Rocky Linux, a business management system used by about a hundred users. The database itself was around 40 GB — nothing extraordinary. But in the data directory there were 180 GB of binary logs. Six months' worth of binlogs that nobody had ever thought to manage.

It's not the first time I've seen this scenario. In fact, I'd say it's one of the most recurring patterns in the tickets I receive. The binary log is one of those MySQL features that works silently, asking nothing — until the disk fills up.

---

## What binary logs actually are

The {{< glossary term="binary-log" >}}binary log{{< /glossary >}} is a sequential record of all events that modify data in the database. Every INSERT, UPDATE, DELETE, every DDL — everything gets written to sequentially numbered binary files: `mysql-bin.000001`, `mysql-bin.000002` and so on.

The name is a bit misleading. It's not a "log" in the syslog or error log sense — it's not meant to be read by a human. It's a structured binary stream that MySQL uses internally for two fundamental purposes:

1. **Replication**: the slave reads the master's binlogs to replicate the same operations
2. **{{< glossary term="pitr" >}}Point-in-time recovery (PITR){{< /glossary >}}**: after restoring a backup, you can "replay" the binlogs to bring data up to a precise moment

Without the binary log, you can't do either. That's why the first instinct — "let's disable binlogs so they don't fill up the disk" — is almost always wrong.

---

## How MySQL generates binlogs

Binary logging is enabled through the `log_bin` parameter. From MySQL 8.0 it's enabled by default — an important change from previous versions where you had to activate it explicitly.

```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server-id = 1
```

MySQL creates a new binlog file under several circumstances:

- When the server starts or restarts
- When the current file reaches the size defined by `max_binlog_size` (default: 1 GB)
- When you run `FLUSH BINARY LOGS`
- When a manual rotation occurs

Each binlog file has an associated index file (`mysql-bin.index`) that tracks all active binlog files. This file is critical: if you corrupt it or edit it by hand, MySQL no longer knows which binlogs exist.

```sql
SHOW BINARY LOGS;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000147 | 1073741824|
| mysql-bin.000148 | 1073741824|
| mysql-bin.000149 | 1073741824|
| ...              |           |
| mysql-bin.000318 |  524288000|
+------------------+-----------+
172 rows in set
```

A hundred and seventy-two files. Each about a gigabyte. The maths checks out: 180 GB of binlogs never purged.

---

## The role in replication

In a master-slave architecture, the binary log is the data transport mechanism. The flow goes like this:

1. The master writes every transaction to the binlog
2. The slave has a thread (I/O thread) that connects to the master and reads the binlogs
3. The slave writes what it receives into its own {{< glossary term="relay-log" >}}relay log{{< /glossary >}}
4. A second thread (SQL thread) on the slave executes events from the {{< glossary term="relay-log" >}}relay log{{< /glossary >}}

This means the binlogs on the master **must remain available until all slaves have read them**. If you delete a binlog that the slave hasn't consumed yet, replication breaks.

Before touching any binlog on a master, the command to run is:

```sql
SHOW REPLICA STATUS\G
-- or, on older versions:
SHOW SLAVE STATUS\G
```

The field you care about is `Relay_Master_Log_File` (or `Source_Log_File` in recent versions): it tells you which binlog the slave is currently reading. All files before that one are safe to remove.

---

## Point-in-time recovery: the other reason binlogs exist

The second use — often underestimated — is point-in-time recovery. The scenario goes like this: you have a backup taken at 3 AM. At 2:30 PM someone runs a wrong `DROP TABLE`. Without binlogs, you restore the backup and lose everything that happened between 3:00 AM and 2:30 PM. With binlogs, you do the restore and then replay the binlogs up to 2:29 PM.

```bash
# Find the DROP TABLE event
mysqlbinlog --start-datetime="2026-03-30 14:00:00" \
            --stop-datetime="2026-03-30 15:00:00" \
            /var/lib/mysql/mysql-bin.000318 | grep -i "DROP"

# Replay binlogs up to the moment before the disaster
mysqlbinlog --stop-datetime="2026-03-30 14:29:00" \
            /var/lib/mysql/mysql-bin.000310 \
            /var/lib/mysql/mysql-bin.000311 \
            ... \
            /var/lib/mysql/mysql-bin.000318 | mysql -u root -p
```

In practice, binlogs are your insurance policy. The backup is the foundation, binlogs cover the delta. Deleting binlogs without a recent backup is like cancelling your insurance the day before a storm.

---

## PURGE BINARY LOGS: the right way to clean up

Back to our server with disk at 95%. The temptation to just do `rm -f mysql-bin.*` is strong. But it's wrong, for two reasons:

1. MySQL doesn't know you deleted the files — the index file still points to binlogs that no longer exist
2. If there's active replication, you risk breaking synchronisation

The correct way is the `PURGE` command:

```sql
-- Remove all binlogs before a specific file
PURGE BINARY LOGS TO 'mysql-bin.000300';

-- Or, remove all binlogs older than a specific date
PURGE BINARY LOGS BEFORE '2026-03-01 00:00:00';
```

`PURGE` does three things that `rm` doesn't:

- Updates the index file
- Checks that the files aren't needed by replication (in theory — but you should check first)
- Removes files in an orderly manner

In our server's case, I first verified there were no slaves:

```sql
SHOW REPLICAS;
-- Empty set
```

No replication. Then I checked which binlog was current:

```sql
SHOW MASTER STATUS;
+------------------+----------+
| File             | Position |
+------------------+----------+
| mysql-bin.000318 | 52428800 |
+------------------+----------+
```

Keeping the last 3 files for safety:

```sql
PURGE BINARY LOGS TO 'mysql-bin.000316';
```

Result: 175 GB freed in a few seconds. Disk usage dropped from 95% to 28%.

---

## Configuring automatic retention

Solving the emergency is one thing. Making sure it doesn't happen again is another. MySQL offers two parameters for automatic retention management:

### `expire_logs_days` (legacy)

```ini
[mysqld]
expire_logs_days = 14
```

Automatically removes binlogs older than 14 days. Simple but coarse — granularity is in days only.

### `binlog_expire_logs_seconds` (MySQL 8.0+)

```ini
[mysqld]
binlog_expire_logs_seconds = 1209600   # 14 days in seconds
```

Same logic, but with per-second granularity. From MySQL 8.0, this parameter takes priority over `expire_logs_days`. If you set both, `binlog_expire_logs_seconds` wins.

The question I always get asked is: "How many days of retention?"

It depends. But here are my practical rules:

| Scenario | Recommended retention |
|----------|----------------------|
| Standalone server, daily backup | 7 days |
| Master with replica, daily backup | 7-14 days |
| Master with slow or remote replica | 14-30 days |
| Regulated environments (finance, healthcare) | 30-90 days, with archiving |

The principle is: **binlog retention must cover at least twice the interval between two backups**. If you back up every night, keep at least 2-3 days of binlogs. If you do weekly backups, at least 14 days.

In our server's case, no retention had been set. The MySQL 8.0 default is 30 days — but that value had been overridden to 0 (no expiry) in a custom `my.cnf` by someone who "wanted to keep everything for safety". The irony: the safety they wanted to guarantee was about to crash the server by filling up the disk.

---

## The three binlog formats: STATEMENT, ROW, MIXED

Not all binlogs are created equal. MySQL supports three recording formats, and the choice has real implications.

### STATEMENT

Records the SQL statement as it was executed. Compact, readable, but problematic: functions like `NOW()`, `UUID()`, `RAND()` produce different results on master and slave. Queries with `LIMIT` without `ORDER BY` can produce non-deterministic results.

```sql
SET binlog_format = 'STATEMENT';
```

### ROW

Records the change at row level — before and after. Heavier in terms of space, but 100% deterministic. If you update 10,000 rows, the binlog contains 10,000 before/after images. Large, but safe.

```sql
SET binlog_format = 'ROW';
```

### MIXED

MySQL decides case by case: uses STATEMENT when it's safe, automatically switches to ROW when it detects non-deterministic operations.

```sql
SET binlog_format = 'MIXED';
```

My advice: **use ROW**. It's been the default since MySQL 5.7.7, it's what Galera Cluster requires, it's what all modern replication tools expect. STATEMENT is a legacy from the past, MIXED is a compromise that adds complexity without real benefit.

The only case where ROW becomes a problem is when you do massive operations — an `UPDATE` on millions of rows generates a huge binlog because it contains the before and after of every row. In those cases, the solution isn't to change format, but to break the operation into batches:

```sql
-- Instead of this (generates a massive binlog):
UPDATE orders SET status = 'archived' WHERE order_date < '2025-01-01';

-- Better like this (batches of 10,000):
UPDATE orders SET status = 'archived'
WHERE order_date < '2025-01-01' AND status != 'archived'
LIMIT 10000;
-- Repeat until 0 rows affected
```

---

## `mysqlbinlog`: reading binlogs when you need to

The {{< glossary term="mysqlbinlog" >}}`mysqlbinlog`{{< /glossary >}} command-line tool is the only way to inspect the contents of binlog files. It's used in two scenarios: debugging replication problems and point-in-time recovery.

```bash
# Read a binlog in human-readable format
mysqlbinlog /var/lib/mysql/mysql-bin.000318

# Filter by time range
mysqlbinlog --start-datetime="2026-03-30 10:00:00" \
            --stop-datetime="2026-03-30 11:00:00" \
            /var/lib/mysql/mysql-bin.000318

# Filter by specific database
mysqlbinlog --database=gestionale /var/lib/mysql/mysql-bin.000318

# If format is ROW, decode into readable SQL
mysqlbinlog --verbose /var/lib/mysql/mysql-bin.000318
```

With ROW format, without `--verbose` you'll only see binary blobs. With `--verbose` you get rows in commented pseudo-SQL format — not pretty, but readable.

---

## The principle: manage binlogs, don't disable them

Every now and then someone suggests solving the problem "at the root" by disabling binlogs:

```ini
# DO NOT DO THIS in production
skip-log-bin
```

Yes, it solves the disk problem. But it eliminates:

- The ability to set up replication in the future
- Point-in-time recovery
- The ability to analyse what happened in the database after an incident
- Compatibility with {{< glossary term="cdc" >}}CDC (Change Data Capture){{< /glossary >}} tools like Debezium

Binlogs are not a problem. **Unmanaged** binlogs are a problem. The difference is a configuration parameter and a weekly check. On the server I fixed, the final configuration was:

```ini
[mysqld]
log_bin = /var/lib/mysql/mysql-bin
server-id = 1
binlog_format = ROW
binlog_expire_logs_seconds = 604800    # 7 days
max_binlog_size = 512M
```

A `max_binlog_size` of 512 MB instead of the default 1 GB — smaller files are easier to manage, transfer and purge. Seven-day retention with daily backup ensures complete PITR coverage with predictable disk usage.

---

## Post-intervention check-up

Before closing the ticket, I added a couple of queries to the client's monitoring system:

```sql
-- Space used by binlogs
SELECT
    COUNT(*) AS num_files,
    ROUND(SUM(file_size) / 1024 / 1024 / 1024, 2) AS total_gb
FROM information_schema.BINARY_LOGS;   -- MySQL 8.0+ / Performance Schema

-- Or, for all versions:
SHOW BINARY LOGS;
-- and sum manually or via script
```

```bash
# Alert if binlogs exceed 20 GB
#!/bin/bash
BINLOG_SIZE=$(mysql -u monitor -p'pwd' -Bse \
  "SELECT ROUND(SUM(file_size)/1024/1024/1024,2) FROM performance_schema.binary_log_status" 2>/dev/null)

# Fallback for versions without performance_schema.binary_log_status
if [ -z "$BINLOG_SIZE" ]; then
    BINLOG_SIZE=$(du -sh /var/lib/mysql/mysql-bin.* 2>/dev/null | \
      awk '{sum+=$1} END {printf "%.2f", sum/1024}')
fi

if (( $(echo "$BINLOG_SIZE > 20" | bc -l) )); then
    echo "WARNING: binlog size ${BINLOG_SIZE} GB"
fi
```

Three weeks after the intervention, the binlogs were using 8 GB — exactly within the predicted window. The disk never went above 45%.

The binlog is like engine oil: you never think about it until the warning light comes on. The difference is that the engine warns you. MySQL doesn't — it keeps writing binlogs as long as the filesystem responds. When it stops responding, it's too late to wonder why you hadn't set up retention.

------------------------------------------------------------------------

## Glossary

**[Binary log](/en/glossary/binary-log/)** — MySQL's sequential binary record that tracks all data modifications (INSERT, UPDATE, DELETE, DDL), used for replication and point-in-time recovery. Enabled by default since MySQL 8.0.

**[PITR](/en/glossary/pitr/)** — Point-in-Time Recovery: a restore technique that combines a full backup with binary logs to bring the database back to any moment in time, not just the backup time.

**[Relay log](/en/glossary/relay-log/)** — Intermediate log file on a MySQL slave that receives events from the master's binary log before they are executed locally by the SQL thread.

**[CDC](/en/glossary/cdc/)** — Change Data Capture: a technique for intercepting data changes in real time by reading transaction logs. Tools like Debezium read MySQL binary logs to propagate changes to external systems.

**[mysqlbinlog](/en/glossary/mysqlbinlog/)** — MySQL command-line utility for reading, filtering and replaying the contents of binary log files. Essential for point-in-time recovery and replication debugging.
