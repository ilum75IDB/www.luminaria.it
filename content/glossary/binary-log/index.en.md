---
title: "Binary log"
description: "MySQL's sequential binary record that tracks all data modifications, used for replication and point-in-time recovery."
translationKey: "glossary_binary-log"
aka: "binlog"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/binary-log-mysql"
---

The **binary log** (or binlog) is a sequential binary-format record where MySQL writes all events that modify data: INSERT, UPDATE, DELETE and DDL operations. Files are numbered progressively (`mysql-bin.000001`, `mysql-bin.000002`, etc.) and managed through an index file.

## How it works

From MySQL 8.0, binary logging is enabled by default via the `log_bin` parameter. MySQL creates a new binlog file when the server starts, when the current file reaches `max_binlog_size`, or when `FLUSH BINARY LOGS` is executed. It supports three recording formats: STATEMENT (records SQL statements), ROW (records row-level changes) and MIXED (automatic choice).

## What it's for

The binary log serves two fundamental purposes:

- **Replication**: in a master-slave architecture, the slave reads the master's binlogs to replicate the same operations
- **Point-in-time recovery**: after restoring a backup, binlogs allow replaying changes up to a precise moment

## When to use it

Binary logging is active by default on any MySQL 8.0+ installation. Active management (retention, purge, space monitoring) is necessary to prevent accumulated files from filling up the disk. The `PURGE BINARY LOGS` command is the correct way to remove obsolete files.
