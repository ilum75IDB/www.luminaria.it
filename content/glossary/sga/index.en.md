---
title: "SGA"
description: "System Global Area — Oracle Database's shared memory area containing buffer cache, shared pool, redo log buffer and other structures critical for performance."
translationKey: "glossary_sga"
aka: "System Global Area"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

The **SGA** (System Global Area) is Oracle Database's main shared memory area. It contains fundamental data structures: buffer cache (data pages read from disk), shared pool (execution plans and data dictionary), redo log buffer, and large pool.

## How it works

SGA size is controlled by the `SGA_TARGET` or `SGA_MAX_SIZE` parameter. Oracle allocates the SGA at instance startup in the operating system's shared memory. The Linux kernel parameters `shmmax` and `shmall` must be sized to allow complete SGA allocation.

## What it's for

All database read and write activity passes through the SGA. An efficient buffer cache avoids physical disk reads. A well-sized shared pool avoids query re-parsing. The SGA is the heart of Oracle performance — and must reside in Huge Pages to maximize efficiency.

## Why it matters

An SGA not allocated in Huge Pages means millions of Page Table entries and constant TLB overflow. The result is latch free waits, library cache contention, and high CPU. Configuring Huge Pages and the `memlock unlimited` parameter for the oracle user is the prerequisite for any serious tuning.
