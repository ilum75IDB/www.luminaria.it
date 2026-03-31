---
title: "Single-primary"
description: "MySQL Group Replication mode where only one node accepts writes while the others are read-only with automatic failover."
translationKey: "glossary_single_primary"
aka: "Single-primary mode"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
---

**Single-primary** is the most common operating mode in MySQL Group Replication, where only one node in the cluster — the primary — accepts write operations. The other nodes (secondaries) are read-only (`read_only=ON`, `super_read_only=ON`) and receive changes through the group's synchronous replication.

## How it works

The parameter `group_replication_single_primary_mode=ON` enables this mode. The primary is the only node with `read_only=OFF`. If the primary is stopped or becomes unreachable, the cluster triggers an automatic election and one of the secondaries becomes the new primary within seconds.

## Why it's used

Single-primary mode avoids the concurrent write conflicts typical of multi-primary setups. In production, most MySQL clusters use this mode because it's more predictable: applications write to a single endpoint, replication is linear, and debugging is simpler.

## What can go wrong

When the primary is stopped for maintenance, the cluster performs an automatic failover. During those seconds, active connections may be dropped and in-flight transactions may fail. It's a brief disruption but it must be communicated. The practical rule: in a maintenance intervention on a single-primary cluster, secondaries are touched first, the primary last.
