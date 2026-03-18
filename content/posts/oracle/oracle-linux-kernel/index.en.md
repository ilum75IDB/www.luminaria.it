---
title: "Oracle on Linux: the kernel parameters nobody configures"
description: "A client with Oracle 19c on Linux and disappointing performance. Default installation, no tuning. Huge Pages, semaphores, I/O scheduler, THP and security limits: everything that was missing — with the before-and-after numbers."
date: "2026-02-24T10:00:00+01:00"
draft: false
translationKey: "oracle_linux_kernel"
tags: ["oracle", "linux", "kernel", "tuning", "hugepages", "performance"]
categories: ["oracle"]
image: "oracle-linux-kernel.cover.jpg"
---

The client was a logistics company running Oracle 19c Enterprise Edition on Oracle Linux 8. Sixty concurrent users, a custom ERP application, about 400 GB of data. The server was a Dell PowerEdge with 128 GB of RAM and 32 cores.

The complaints were vague but persistent: "The system is slow." "Morning queries take twice as long as two months ago." "Every now and then everything freezes for a few seconds."

When I logged into the server, the first thing I checked was not the database. It was the operating system.

``` bash
cat /proc/meminfo | grep -i huge
sysctl vm.nr_hugepages
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Result: zero Huge Pages configured, Transparent Huge Pages active, kernel parameters all at default values. The Oracle installation had been done with the wizard, the operating system had never been touched.

There was the problem. It wasn't Oracle. It was Linux that hadn't been prepared for Oracle.

------------------------------------------------------------------------

## 🔍 The diagnosis

Before changing anything, I measured the current state. You need numbers, not impressions.

``` bash
# SGA status
sqlplus -s / as sysdba <<SQL
SELECT name, value/1024/1024 AS mb
FROM   v$sgainfo
WHERE  name IN ('Maximum SGA Size', 'Free SGA Memory Available');
SQL

# System memory usage
free -h

# Current kernel parameters
sysctl -a | grep -E "kernel.sem|kernel.shm|vm.nr_hugepages|vm.swappiness"

# I/O scheduler in use
cat /sys/block/sda/queue/scheduler

# Oracle user limits
su - oracle -c "ulimit -a"
```

Here is what I found:

| Parameter | Current value | Recommended value |
|---|---|---|
| SGA Target | 64 GB | 64 GB (ok) |
| vm.nr_hugepages | 0 | 33280 |
| Transparent Huge Pages | always | never |
| vm.swappiness | 60 | 1 |
| kernel.shmmax | 33554432 (32 MB) | 68719476736 (64 GB) |
| kernel.shmall | 2097152 | 16777216 |
| kernel.sem | 250 32000 100 128 | 250 32000 100 256 |
| I/O scheduler | mq-deadline | deadline (ok) |
| oracle nofile | 1024 | 65536 |
| oracle nproc | 4096 | 16384 |
| oracle memlock | 65536 KB | unlimited |

Nearly everything was wrong. Not by mistake — by omission. Nobody had bothered to configure the operating system after installation.

------------------------------------------------------------------------

## 📦 Huge Pages: the parameter that changes everything

Huge Pages are the single most impactful parameter for Oracle on Linux. And they are also the one most often ignored.

### Why they matter

By default, Linux manages memory in 4 KB pages. A 64 GB SGA means roughly 16.7 million pages. Each page has an entry in the Page Table, and the system must translate virtual addresses to physical ones for each. The CPU's TLB (Translation Lookaside Buffer) can cache only a few thousand translations — the rest is handled by the MMU, which is slow.

Huge Pages are 2 MB pages. The same 64 GB SGA becomes 32,768 pages. The TLB copes, MMU pressure drops, performance improves.

### How to configure them

I calculated the number of Huge Pages needed:

``` bash
# SGA = 64 GB = 65536 MB
# Each Huge Page = 2 MB
# Pages needed = 65536 / 2 = 32768
# Adding 1.5% margin → 33280

echo "vm.nr_hugepages = 33280" >> /etc/sysctl.d/99-oracle.conf
sysctl -p /etc/sysctl.d/99-oracle.conf
```

Verification:

``` bash
grep -i huge /proc/meminfo
```

Expected output:

```
HugePages_Total:   33280
HugePages_Free:    33280
HugePages_Rsvd:        0
Hugepagesize:       2048 kB
```

After restarting the Oracle instance, the SGA gets allocated in Huge Pages:

```
HugePages_Total:   33280
HugePages_Free:      512
HugePages_Rsvd:      480
```

The difference is measurable: `latch free` waits and `library cache` contention drop dramatically.

------------------------------------------------------------------------

## 🧱 Shared memory and semaphores

Oracle uses kernel shared memory for the SGA. If the limits are too low, the instance cannot allocate the requested memory — or worse, fragments the allocation.

``` bash
cat >> /etc/sysctl.d/99-oracle.conf << 'SYSCTL'
# Shared memory
kernel.shmmax = 68719476736
kernel.shmall = 16777216
kernel.shmmni = 4096

# Semaphores: SEMMSL SEMMNS SEMOPM SEMMNI
kernel.sem = 250 32000 100 256
SYSCTL

sysctl -p /etc/sysctl.d/99-oracle.conf
```

| Parameter | Meaning | Value |
|---|---|---|
| shmmax | Maximum size of a single shared memory segment | 64 GB |
| shmall | Total pages of shared memory allocatable | 64 GB in 4K pages |
| shmmni | Maximum number of shared memory segments | 4096 |
| sem | SEMMSL, SEMMNS, SEMOPM, SEMMNI | 250 32000 100 256 |

These are not magic numbers. They are sized for the database's SGA. If the SGA changes, the parameters need recalculating.

------------------------------------------------------------------------

## 💾 I/O Scheduler

The default on RHEL/Oracle Linux 8 with NVMe devices is `none` or `mq-deadline`. For traditional SAS/SATA disks, the default may be `bfq` or `cfq`.

For Oracle, the recommendation is `deadline` (or `mq-deadline` on newer kernels):

``` bash
# Check current setting
cat /sys/block/sda/queue/scheduler

# If not deadline/mq-deadline, set it
echo "deadline" > /sys/block/sda/queue/scheduler

# Make it permanent via GRUB
grubby --update-kernel=ALL --args="elevator=deadline"
```

`cfq` (Completely Fair Queuing) is designed for desktop workloads — it distributes I/O fairly across processes. But Oracle doesn't need fairness: it needs I/O requests served in the order that minimises seeks. `deadline` does exactly that.

------------------------------------------------------------------------

## 🚫 Disabling Transparent Huge Pages

This is the most insidious parameter. Transparent Huge Pages (THP) is a kernel feature that **sounds** like a good idea: the kernel automatically promotes normal pages to huge pages.

For Oracle it is a disaster. The `khugepaged` process works in the background to compact pages, causing unpredictable latency spikes — those "freezes for a few seconds" the client had been complaining about.

Oracle says it explicitly in the documentation: **disable THP**.

``` bash
# Check current state
cat /sys/kernel/mm/transparent_hugepage/enabled
# Typical output: [always] madvise never

# Disable at runtime
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Make permanent via GRUB
grubby --update-kernel=ALL --args="transparent_hugepage=never"
```

After reboot, verify:

``` bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# Expected output: always madvise [never]
```

The difference is stark: random micro-freezes disappear.

------------------------------------------------------------------------

## 🔒 Security limits

The `oracle` user needs elevated limits on open file descriptors, processes and lockable memory. Linux defaults are designed for interactive users, not for software that manages hundreds of simultaneous connections.

``` bash
cat >> /etc/security/limits.d/99-oracle.conf << 'LIMITS'
oracle   soft   nofile    65536
oracle   hard   nofile    65536
oracle   soft   nproc     16384
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   soft   memlock   unlimited
oracle   hard   memlock   unlimited
LIMITS
```

| Limit | Default | Recommended | Why |
|---|---|---|---|
| nofile | 1024 | 65536 | Oracle opens a file descriptor for every datafile, redo log, archive log |
| nproc | 4096 | 16384 | Each Oracle process is a separate OS process |
| memlock | 65536 KB | unlimited | Required for locking the SGA into Huge Pages |
| stack | 8192 KB | 10240-32768 KB | Deep recursive PL/SQL can exhaust the stack |

The `memlock unlimited` setting is critical: without it, Oracle cannot lock the SGA into Huge Pages, making the earlier configuration pointless.

------------------------------------------------------------------------

## ⚡ Swappiness

The default value of `vm.swappiness` is 60. That means Linux starts swapping when memory pressure is still low. For a dedicated database server, this is unacceptable: you want the SGA to stay in RAM, always.

``` bash
echo "vm.swappiness = 1" >> /etc/sysctl.d/99-oracle.conf
sysctl -p /etc/sysctl.d/99-oracle.conf
```

Not zero — one. A value of zero completely disables swap, which can trigger the OOM killer under extreme pressure. A value of one tells the kernel: "Only swap when there is truly no alternative."

------------------------------------------------------------------------

## 📊 Before and after

After applying all configurations and restarting the Oracle instance, I ran the measurements again.

| Metric | Before | After | Change |
|---|---|---|---|
| SGA in Huge Pages | No | Yes | — |
| Library cache hit ratio | 92.3% | 99.7% | +7.4% |
| Buffer cache hit ratio | 94.1% | 99.2% | +5.1% |
| Average wait time (db file sequential read) | 8.2 ms | 1.4 ms | -83% |
| Random micro-freezes (>1s) | 5-8 per day | 0 | -100% |
| Average morning batch time | 47 min | 22 min | -53% |
| Average CPU utilisation | 78% | 41% | -47% |
| Swap used | 3.2 GB | 0 MB | -100% |

The numbers speak for themselves. Same machine, same database, same workload. The only difference: the operating system was configured to do its job.

------------------------------------------------------------------------

## 📋 Final checklist

For those who want an operational summary, here is the complete checklist:

``` bash
# /etc/sysctl.d/99-oracle.conf
vm.nr_hugepages = 33280
vm.swappiness = 1
kernel.shmmax = 68719476736
kernel.shmall = 16777216
kernel.shmmni = 4096
kernel.sem = 250 32000 100 256
```

``` bash
# /etc/security/limits.d/99-oracle.conf
oracle   soft   nofile    65536
oracle   hard   nofile    65536
oracle   soft   nproc     16384
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   soft   memlock   unlimited
oracle   hard   memlock   unlimited
```

``` bash
# GRUB
grubby --update-kernel=ALL --args="transparent_hugepage=never elevator=deadline"
```

Ten minutes of configuration. No hardware cost. No additional licences.

But nobody does it, because the wizard doesn't ask, the documentation is buried in an MOS note, and the system "works without it." It works. Poorly. And the blame always falls on Oracle, never on the fact that nobody prepared the ground.

A database is only as good as the operating system it runs on. And an operating system left at defaults is an operating system working against you.

------------------------------------------------------------------------

## Glossary

**[Huge Pages](/en/glossary/huge-pages/)** — 2 MB memory pages (instead of the standard 4 KB) that drastically reduce MMU and TLB pressure, improving Oracle performance on Linux.

**[THP](/en/glossary/thp/)** — Transparent Huge Pages — Linux kernel feature that automatically promotes normal pages to huge pages, but causes unpredictable latencies and must be disabled for Oracle.

**[SGA](/en/glossary/sga/)** — System Global Area — Oracle Database's shared memory area containing buffer cache, shared pool, redo log buffer and other structures critical for performance.

**[I/O Scheduler](/en/glossary/io-scheduler/)** — Linux kernel component that decides the order in which I/O requests are sent to disk, with direct impact on database performance.

**[Swappiness](/en/glossary/swappiness/)** — Linux kernel parameter (vm.swappiness) controlling the system's propensity to move memory pages to swap, critical for database servers.
