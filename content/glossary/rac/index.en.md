---
title: "RAC"
description: "Real Application Clusters — Oracle technology that allows multiple instances to simultaneously access the same database, providing high availability and scalability."
translationKey: "glossary_rac"
aka: "Real Application Clusters"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**RAC** (Real Application Clusters) is Oracle's technology that allows multiple database instances to simultaneously access the same shared storage. If a node fails, the others continue serving requests without interruption — failover is transparent to applications.

## How it works

A RAC cluster consists of two or more servers (nodes) connected via a high-speed private network (interconnect) and shared storage (typically ASM — Automatic Storage Management). Each node runs its own Oracle instance, but all access the same datafiles.

The **Cache Fusion** protocol manages data coherence across nodes: when a block modified by one node is needed by another, it's transferred directly via the interconnect without going through disk.

## High availability

If a node goes down, active sessions are automatically transferred to the remaining nodes via **TAF** (Transparent Application Failover) or **Application Continuity**. Failover time depends on configuration but is typically in the order of seconds.

## Licensing implications

RAC is an Enterprise Edition option with significant licensing costs. During cloud migration, RAC license counting is one of the most sensitive aspects: on OCI with BYOL, on-premises licenses are reused; on other cloud providers, the cost can multiply.

## When it's really needed

RAC is justified when automatic failover high availability and horizontal scalability are required. For environments with few users or standard uptime requirements, a single node with Data Guard is often a simpler and less expensive solution.
