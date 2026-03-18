---
title: "I/O Scheduler"
description: "Linux kernel component that decides the order in which I/O requests are sent to disk, with direct impact on database performance."
translationKey: "glossary_io-scheduler"
aka: "Linux I/O Scheduler"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

The **I/O Scheduler** is the Linux kernel component that manages the queue of read and write requests to block devices (disks). It decides the execution order of requests to optimize throughput and minimize latency.

## How it works

Linux offers several schedulers: `cfq` (Completely Fair Queuing, for desktops), `deadline`/`mq-deadline` (for servers and databases), `noop`/`none` (for SSD/NVMe). For Oracle the recommendation is `deadline`, which serves requests minimizing disk seeks. It is configured via `/sys/block/sdX/queue/scheduler` and made permanent via GRUB.

## What it's for

The default `cfq` distributes I/O equally among processes — ideal for a desktop, terrible for a database that needs priority on critical I/O requests. `deadline` ensures no request stays in queue too long, reducing `db file sequential read` latency.

## What can go wrong

Leaving the default (`cfq` or `bfq` on some systems) means Oracle competes for I/O with all other system processes. On a dedicated database server this is wasteful: the database should have absolute priority on disk operations.
