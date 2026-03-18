---
title: "SST"
description: "State Snapshot Transfer — Galera Cluster mechanism for transferring a complete data copy to a node joining the cluster."
translationKey: "glossary_sst"
aka: "State Snapshot Transfer"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**SST** (State Snapshot Transfer) is the mechanism by which a Galera node joining the cluster (or one that has been offline too long) receives a complete copy of the entire dataset from a donor node.

## How it works

When a node joins the cluster and the gap of missing transactions exceeds the gcache size, the cluster initiates an SST. The donor node creates a full snapshot of the database and transfers it to the receiving node. Available methods include: `mariabackup` (doesn't block the donor), `rsync` (fast but blocks the donor for reads), and `mysqldump` (slow and blocking).

## What it's for

SST is essential for two scenarios: adding a new node to the cluster (first join) and recovering a node that has been offline so long that missing transactions are no longer available in the donor's gcache.

## When to use it

SST is triggered automatically by Galera when needed. The SST method choice (`wsrep_sst_method`) is made during configuration. In production, `mariabackup` is the recommended choice because it doesn't block the donor node, avoiding cluster degradation during the transfer.
