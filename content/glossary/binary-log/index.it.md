---
title: "Binary log"
description: "Registro binario sequenziale di MySQL che traccia tutte le modifiche ai dati, usato per la replica e il point-in-time recovery."
translationKey: "glossary_binary-log"
aka: "binlog"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/binary-log-mysql"
---

Il **binary log** (o binlog) è un registro sequenziale in formato binario in cui MySQL scrive tutti gli eventi che modificano i dati: INSERT, UPDATE, DELETE e operazioni DDL. I file sono numerati progressivamente (`mysql-bin.000001`, `mysql-bin.000002`, ecc.) e gestiti tramite un file indice.

## Come funziona

Da MySQL 8.0 il binary log è abilitato di default tramite il parametro `log_bin`. MySQL crea un nuovo file binlog quando il server si avvia, quando il file corrente raggiunge `max_binlog_size`, o quando si esegue `FLUSH BINARY LOGS`. Supporta tre formati di registrazione: STATEMENT (registra le istruzioni SQL), ROW (registra le modifiche riga per riga) e MIXED (scelta automatica).

## A cosa serve

Il binary log ha due funzioni fondamentali:

- **Replica**: in un'architettura master-slave, lo slave legge i binlog del master per replicare le stesse operazioni
- **Point-in-time recovery**: dopo un restore da backup, i binlog permettono di riapplicare le modifiche fino a un momento preciso

## Quando si usa

Il binary log è attivo di default su qualsiasi installazione MySQL 8.0+. La gestione attiva (retention, purge, monitoraggio dello spazio) è necessaria per evitare che i file accumulati riempiano il disco. Il comando `PURGE BINARY LOGS` è il modo corretto per eliminare i file obsoleti.
