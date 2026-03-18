---
title: "IST"
description: "Incremental State Transfer — Galera Cluster mechanism for transferring only missing transactions to a node rejoining the cluster."
translationKey: "glossary_ist"
aka: "Incremental State Transfer"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**IST** (Incremental State Transfer) is the mechanism by which a Galera node rejoining the cluster after a brief absence receives only the missing transactions, without having to download the entire dataset.

## How it works

When a node reconnects to the cluster, the donor checks whether the missing transactions are still available in its gcache (Galera cache). If the gap is covered by the gcache, an IST is performed: only the missing transactions are sent to the node, which applies them and returns to Synced state. If the gap exceeds the gcache, Galera falls back to a full SST.

## What it's for

IST makes a node's return to the cluster much faster than a full SST. A node that has been offline for a few minutes or hours can become operational again in seconds, with no impact on cluster performance.

## When to use it

IST is triggered automatically when conditions allow it. The gcache size (`gcache.size`) determines how many transactions the cluster can keep in memory to support IST. A larger gcache allows longer node downtime without requiring an SST.
