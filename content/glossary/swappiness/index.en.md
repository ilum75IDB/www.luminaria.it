---
title: "Swappiness"
description: "Linux kernel parameter (vm.swappiness) controlling the system's propensity to move memory pages to swap, critical for database servers where the SGA must stay in RAM."
translationKey: "glossary_swappiness"
aka: "vm.swappiness"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

**Swappiness** (`vm.swappiness`) is a Linux kernel parameter that controls how aggressively the system moves memory pages from RAM to swap on disk. The value ranges from 0 (swap only in extreme cases) to 100 (aggressive swapping). The default is 60.

## How it works

With the default value of 60, Linux starts swapping when memory pressure is still relatively low. For a dedicated database server, this is unacceptable: the SGA must stay in RAM, always. The recommended value for Oracle is 1 — not 0, which would completely disable swap and could trigger the OOM killer.

## What it's for

The value 1 tells the kernel: "Swap only if there is truly no alternative." This ensures the SGA and Oracle's critical structures remain in physical memory, avoiding swap reads (orders of magnitude slower than RAM) during query execution.

## Why it matters

With swappiness at 60, a server with 128 GB of RAM and a 64 GB SGA may start swapping parts of the SGA even with 20-30 GB of free RAM. The result is unpredictably degraded performance, with latency spikes that look like application problems but are actually the OS moving memory to disk.
