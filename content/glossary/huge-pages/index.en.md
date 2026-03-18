---
title: "Huge Pages"
description: "2 MB memory pages (instead of the standard 4 KB) that drastically reduce MMU and TLB pressure, improving Oracle performance on Linux."
translationKey: "glossary_huge-pages"
aka: "HugePages / Large Pages"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

**Huge Pages** are 2 MB memory pages, compared to Linux's standard 4 KB. For a 64 GB Oracle SGA, switching from 4 KB pages (16.7 million pages) to 2 MB Huge Pages (32,768 pages) reduces the number of Page Table entries by a factor of 500.

## How it works

They are configured via the kernel parameter `vm.nr_hugepages` in `/etc/sysctl.d/`. The required number is calculated by dividing the SGA size by 2 MB and adding a 1.5% margin. After restarting the Oracle instance, the SGA is allocated in Huge Pages, verifiable from `/proc/meminfo`.

## What it's for

They reduce pressure on the CPU's TLB (Translation Lookaside Buffer), which can cache only a few thousand address translations. With normal pages, the TLB constantly overflows and the MMU must handle millions of translations — with measurable impact on latch free waits and library cache contention.

## Why it matters

It is the single most impactful parameter for Oracle on Linux, and the one most often ignored. The installation wizard doesn't configure it, the documentation is in an MOS note, and the system "works without it." But the before/after metrics speak clearly: library cache hit ratio from 92% to 99.7%, CPU from 78% to 41%.
