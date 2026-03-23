---
title: "Transport Lag"
description: "The delay in transmitting redo logs from the primary database to the standby in a Data Guard configuration. A critical indicator of replication health."
translationKey: "glossary_transport_lag"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**Transport lag** is the delay between when the primary database generates a redo log and when that redo log is received by the standby database in an Oracle Data Guard configuration. It's one of the most important indicators for assessing replication health.

## How it's measured

Transport lag is monitored via a query on the `V$DATAGUARD_STATS` view or through Data Guard Broker:

    DGMGRL> SHOW DATABASE 'standby_db' 'TransportLagTarget';

The value is expressed in time format (e.g., `+00 00:00:03` = 3 seconds of delay). A transport lag of a few seconds is normal in Maximum Performance mode; a lag that consistently grows indicates a bandwidth or redo generate rate problem.

## Difference from Apply Lag

| Metric | What it measures |
|--------|-----------------|
| **Transport Lag** | Delay in transmitting redo from primary to standby |
| **Apply Lag** | Delay in applying redo on the standby after reception |

Transport lag depends on the network (bandwidth, latency); apply lag depends on standby resources (CPU, I/O). In cross-site migrations, transport lag is the most common bottleneck.

## Impact in migrations

During a cross-site Data Guard migration, transport lag must be carefully monitored during peak load phases (nightly batches, activity spikes). A redo generate rate that exceeds VPN capacity produces a growing transport lag. Before cutover, the transport lag must be close to zero to ensure the switchover happens without data loss.
