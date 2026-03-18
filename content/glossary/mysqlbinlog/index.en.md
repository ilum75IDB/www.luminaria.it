---
title: "mysqlbinlog"
description: "MySQL command-line utility for reading, filtering and replaying the contents of binary log files."
translationKey: "glossary_mysqlbinlog"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**mysqlbinlog** is the command-line utility shipped with MySQL for reading and decoding the contents of binary log files. It is the only tool capable of converting the binary format of binlogs into readable output or re-executable SQL statements.

## How it works

mysqlbinlog reads binlog files and produces text-format output. It supports several filters:

- **By time range**: `--start-datetime` and `--stop-datetime` to limit output to a time window
- **By database**: `--database` to filter events for a specific database
- **By position**: `--start-position` and `--stop-position` to select specific events

With ROW format, the `--verbose` flag decodes row-level changes into commented pseudo-SQL format, otherwise the output is an unreadable binary blob.

## What it's for

mysqlbinlog is used in two main scenarios:

- **Point-in-time recovery**: extracting and replaying events from backup to the desired moment, piping output directly into the mysql client
- **Replication debugging**: analysing events to understand what was replicated, identifying problematic transactions or reconstructing the sequence of operations that caused an issue

## When to use it

mysqlbinlog is essential whenever you need to inspect what happened in the database after an incident, or when performing a point-in-time recovery. It requires access to binlog files on the server filesystem or the ability to connect to the server with `--read-from-remote-server`.
