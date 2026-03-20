---
title: "Exchange Partition"
description: "An Oracle DDL operation that instantly swaps data segments between a non-partitioned table and a partition, without physically moving any data."
translationKey: "glossary_exchange-partition"
aka: "Partition exchange"
articles:
  - "/posts/data-warehouse/partitioning-dwh"
---

**Exchange Partition** is an Oracle DDL operation that allows you to instantly swap the contents of a partition with those of a non-partitioned table. Not a single byte of data is moved — the operation only modifies pointers in the data dictionary.

## How it works

The `ALTER TABLE ... EXCHANGE PARTITION ... WITH TABLE ...` command modifies metadata in the data dictionary so that the physical segments of the partition and the staging table swap ownership. The staging table becomes the partition and vice versa. The operation takes less than a second regardless of data volume, because it involves no physical data movement.

## What it's for

In data warehouses, exchange partition is the primary tool for bulk data loading. The typical process is: the ETL loads data into a staging table, builds indexes, validates the data, and then executes the exchange with the target partition. During the exchange, queries on other partitions continue working without interruption.

## What can go wrong

The `WITHOUT VALIDATION` clause skips the check that the staging table's data actually falls within the partition's range — it speeds up the operation but requires the ETL to guarantee data correctness. If the staging data contains out-of-range dates, they end up in the wrong partition with no error raised. The `INCLUDING INDEXES` clause requires the staging table to have indexes with the same structure as the partitioned table's local indexes.
