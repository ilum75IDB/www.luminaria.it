---
title: "PITR"
description: "Point-in-Time Recovery — a restore technique that allows bringing a database back to a precise moment in time, combining backups and transaction logs."
translationKey: "glossary_pitr"
aka: "Point-in-Time Recovery"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**PITR** (Point-in-Time Recovery) is a restore technique that allows bringing a database back to any moment in time, not just the moment of the backup. It relies on combining a full backup with transaction logs (binary logs in MySQL, WAL in PostgreSQL, redo logs in Oracle).

## How it works

The process has two phases:

1. **Backup restore**: the database is restored to the last available backup
2. **Log replay**: transaction logs are replayed from the backup moment up to the desired point in time, excluding the event that caused the problem

In MySQL, the `mysqlbinlog` tool extracts events from binary logs and replays them on the restored database.

## What it's for

PITR is essential when a human error occurs (DROP TABLE, DELETE without WHERE, wrong mass UPDATE) and the database needs to be restored to the state immediately before the error, without losing the hours of work between the last backup and the incident.

## When to use it

PITR requires binary logging to be active and binlog files not to have been deleted. Binlog retention should cover at least twice the interval between two consecutive backups to guarantee complete PITR coverage.
