---
title: "CDC"
description: "Change Data Capture — a technique for intercepting and propagating data changes in real time, often based on reading transaction logs."
translationKey: "glossary_cdc"
aka: "Change Data Capture"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**CDC** (Change Data Capture) is a technique for intercepting data changes (INSERT, UPDATE, DELETE) as they occur and propagating them to other systems in real time or near-real time. Unlike traditional batch approaches (periodic ETL), CDC captures changes continuously and incrementally.

## How it works

The most common approach is **log-based CDC**: an external component reads the database's transaction logs (binary log in MySQL, WAL in PostgreSQL, redo log in Oracle) and converts events into a data stream consumable by other systems. Tools like Debezium, Maxwell and Canal implement this approach for MySQL by reading binary logs directly.

## What it's for

CDC is used for:

- Synchronising data between different databases in real time
- Feeding data warehouses and data lakes with incremental updates
- Populating caches and search indexes (Elasticsearch, Redis)
- Implementing event-driven architectures and microservices

## When to use it

CDC requires binary logging to be active and in ROW format (which records row-level changes). Disabling binary logs or using STATEMENT format eliminates the ability to use CDC tools, making real-time integration with external systems impossible.
