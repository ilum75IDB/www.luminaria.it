---
title: "Relay log"
description: "Intermediate log file on a MySQL slave that receives events from the master's binary log before they are executed locally."
translationKey: "glossary_relay-log"
articles:
  - "/posts/mysql/binary-log-mysql"
---

The **relay log** is an intermediate log file present on the slave in a MySQL replication architecture. It contains events received from the master's binary log, waiting to be executed locally by the slave's SQL thread.

## How it works

MySQL replication flows through the relay log in three phases:

1. The slave's **I/O thread** connects to the master and reads the binary logs
2. Received events are written to the local relay log
3. The slave's **SQL thread** reads events from the relay log and executes them on the local database

This two-thread architecture decouples data reception from data application: the slave can continue receiving events from the master even if local execution is temporarily slower.

## What it's for

The relay log is the mechanism that ensures replication consistency. It acts as a buffer between the master and the local application of events, allowing the slave to handle speed differences without losing data.

## When to use it

The relay log is created automatically when MySQL replication is configured. It doesn't require direct manual management, but its state (current position, potential lag) is visible through `SHOW REPLICA STATUS` and is essential for diagnosing replication lag issues.
