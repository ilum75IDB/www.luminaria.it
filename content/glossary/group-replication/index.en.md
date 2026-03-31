---
title: "Group Replication"
description: "MySQL's native mechanism for synchronous multi-node replication with automatic failover and quorum management."
translationKey: "glossary_group_replication"
aka: "MySQL Group Replication / InnoDB Cluster"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/binary-log-mysql"
---

**Group Replication** is MySQL's native mechanism for creating high-availability clusters with synchronous replication across multiple nodes. Unlike classic replication (asynchronous, master-slave), Group Replication ensures every transaction is confirmed by a majority of nodes before being considered committed.

## How it works

Nodes communicate via a group protocol (GCS — Group Communication System) that handles distributed consensus. Each node maintains a full copy of the data. Transactions are certified by the group: if there are no conflicts, they are applied on all nodes. If a conflict arises, the transaction is rolled back on the originating node.

## Operating modes

MySQL Group Replication supports two modes: **single-primary** (only one node accepts writes, the others are read-only) and **multi-primary** (all nodes accept writes). Single-primary mode is the most common in production because it avoids concurrent write conflicts.

## Why it matters

Group Replication handles failover automatically: if the primary goes down, the cluster elects a new primary from the secondaries within seconds. This makes it suitable for environments that require high availability without manual intervention. It requires a minimum of three nodes to maintain quorum.
