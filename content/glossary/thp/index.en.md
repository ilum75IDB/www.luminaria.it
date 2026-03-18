---
title: "THP"
description: "Transparent Huge Pages — Linux kernel feature that automatically promotes normal pages to huge pages, but causes unpredictable latencies and must be disabled for Oracle."
translationKey: "glossary_thp"
aka: "Transparent Huge Pages"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

**THP** (Transparent Huge Pages) is a Linux kernel feature that automatically promotes 4 KB memory pages to 2 MB in the background, without explicit configuration. Unlike static Huge Pages, they are managed by the `khugepaged` kernel process.

## How it works

When active (default `always`), the kernel attempts to compact normal pages into huge pages in the background. The `khugepaged` process works continuously to find and merge groups of contiguous pages, causing unpredictable micro-freezes during compaction operations.

## Why it matters

For Oracle they are a disaster. Oracle states explicitly in its documentation: disable THP. The "freezes of a few seconds" that users complain about are often caused by `khugepaged`. They are disabled with `echo never > /sys/kernel/mm/transparent_hugepage/enabled` and via GRUB for persistence across reboots.

## What can go wrong

The confusion between Huge Pages (good for Oracle, statically configured) and THP (harmful for Oracle, active by default) is one of the most common mistakes. A DBA who sees "Huge Pages" in the documentation and doesn't disable THP is making things worse instead of better.
