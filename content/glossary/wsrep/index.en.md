---
title: "WSREP"
description: "Write Set Replication — synchronous replication API and protocol used by Galera Cluster to keep cluster nodes aligned in real time."
translationKey: "glossary_wsrep"
aka: "Write Set Replication"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**WSREP** (Write Set Replication) is the API and protocol that Galera Cluster uses for synchronous multi-master replication. Each transaction is captured as a "write set" (a set of row-level changes) and replicated to all cluster nodes before commit.

## How it works

When a node executes a transaction, WSREP intercepts it at commit time, packages it as a write set and sends it to all cluster nodes via the group communication protocol. Each node performs a **certification** process: it verifies that the transaction doesn't conflict with other concurrent transactions. If certification succeeds, all nodes apply the transaction. If it fails, the transaction is rolled back on the originating node.

## What it's for

WSREP ensures all cluster nodes have the same data at all times (synchronous replication). Unlike traditional MySQL asynchronous replication, there is no lag between master and slave: when a transaction is committed on one node, it's already present on all others.

## When to use it

WSREP is activated with the `wsrep_on=ON` parameter in MariaDB/Percona XtraDB Cluster configuration. Status variables starting with `wsrep_` (such as `wsrep_cluster_size`, `wsrep_cluster_status`, `wsrep_flow_control_paused`) are the main indicators for monitoring cluster health.
