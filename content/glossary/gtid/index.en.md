---
title: "GTID"
description: "Global Transaction Identifier — unique identifier assigned to every transaction in MySQL to simplify replication management."
translationKey: "glossary_gtid"
aka: "Global Transaction Identifier"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/mysqldump-mysqlpump-mydumper"
---

**GTID** (Global Transaction Identifier) is a unique identifier automatically assigned to every committed transaction on a MySQL server. The format is `server_uuid:transaction_id` — for example `3E11FA47-71CA-11E1-9E33-C80AA9429562:23`.

## How it works

When GTID is enabled (`gtid_mode = ON`), every transaction receives an identifier that makes it traceable across any server in the replication cluster. The replica knows exactly which transactions it has already executed and which ones it still needs, without having to manually specify binlog positions (file + offset).

The set of all GTIDs executed on a server is stored in the `gtid_executed` variable. When a replica connects to the source, it compares its own `gtid_executed` with the source's to determine which transactions are missing.

## What it's for

GTID radically simplifies MySQL replication management:

- **Automatic failover**: when the source goes down, a replica can become the new source and the other replicas realign automatically
- **Consistency verification**: it's possible to verify whether two servers have executed exactly the same transactions
- **Backup and restore**: tools like mysqldump and mydumper must handle GTIDs correctly to avoid replication conflicts after restore

## When it causes problems

GTIDs require attention during backup and restore operations. If a dump is restored on a server with GTID enabled without correctly setting `--set-gtid-purged`, it can generate conflicts that break the replication chain.
