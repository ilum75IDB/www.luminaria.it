---
title: "Split-brain"
description: "Critical condition in a database cluster where two or more parts operate independently, accepting divergent writes on the same data."
translationKey: "glossary_split-brain"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**Split-brain** is a critical condition that occurs when a database cluster splits into two or more partitions that cannot communicate with each other, and each partition continues accepting writes independently. The result is divergent data impossible to reconcile automatically.

## How it works

In a 3-node cluster, if the network between Node 1 and Nodes 2-3 breaks, without quorum protection both parts could continue accepting writes. When the network is restored, the cluster would find itself with two different versions of the same data. The quorum mechanism prevents this scenario: only the partition with the majority of nodes (quorum) can continue operating.

## What it's for

Understanding split-brain is fundamental for designing reliable database clusters. It's the main reason Galera Cluster requires an odd number of nodes (3, 5, 7) and implements the quorum mechanism. With an even number of nodes, a network partition can split the cluster into two equal halves, neither of which has quorum.

## When to use it

The term split-brain describes a risk to avoid, not a feature to enable. In Galera, protection is automatic: nodes that lose quorum transition to Non-Primary state and refuse writes. The `pc.ignore_quorum` parameter disables this protection, but using it in production is strongly discouraged.
