---
title: "Quorum"
description: "Majority-based consensus mechanism used in database clusters to prevent split-brain and ensure data consistency."
translationKey: "glossary_quorum"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**Quorum** is the minimum number of nodes that must agree for the cluster to be considered operational. In a 3-node cluster, the quorum is 2 (the majority). If one node disconnects, the other two recognise they are the majority and continue operating normally.

## How it works

Galera Cluster uses a group communication protocol that continuously checks how many nodes are reachable. The calculation is simple: quorum = (total nodes / 2) + 1. With 3 nodes the quorum is 2, with 5 nodes it's 3. Nodes that lose quorum transition to Non-Primary state and refuse writes to avoid inconsistencies.

## What it's for

Quorum prevents **split-brain**: the situation where two parts of the cluster operate independently, accepting different writes on the same data. Without quorum, a network partition could lead to divergent data impossible to reconcile automatically.

## When to use it

Quorum is automatically active in any Galera cluster. This is why **three nodes is the minimum in production**: with two nodes, losing one leaves the survivor without quorum and therefore blocked.
