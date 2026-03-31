---
title: "GTID"
description: "Global Transaction Identifier — identificativo univoco assegnato a ogni transazione in MySQL per semplificare la gestione della replica."
translationKey: "glossary_gtid"
aka: "Global Transaction Identifier"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/mysqldump-mysqlpump-mydumper"
---

**GTID** (Global Transaction Identifier) è un identificativo univoco assegnato automaticamente a ogni transazione committata su un server MySQL. Il formato è `server_uuid:transaction_id` — ad esempio `3E11FA47-71CA-11E1-9E33-C80AA9429562:23`.

## Come funziona

Quando il GTID è abilitato (`gtid_mode = ON`), ogni transazione riceve un identificativo che la rende tracciabile su qualsiasi server del cluster di replica. Lo slave sa esattamente quali transazioni ha già eseguito e quali deve ancora ricevere, senza bisogno di specificare manualmente posizioni di binlog (file + offset).

Il set di tutti i GTID eseguiti su un server è memorizzato nella variabile `gtid_executed`. Quando uno slave si connette al master, confronta il proprio `gtid_executed` con quello del master per determinare quali transazioni mancano.

## A cosa serve

Il GTID semplifica radicalmente la gestione della replica MySQL:

- **Failover automatico**: quando il master cade, uno slave può diventare il nuovo master e gli altri slave si riallineano automaticamente
- **Verifica della consistenza**: è possibile verificare se due server hanno eseguito esattamente le stesse transazioni
- **Backup e restore**: strumenti come mysqldump e mydumper devono gestire correttamente i GTID per evitare conflitti di replica dopo il restore

## Quando crea problemi

I GTID richiedono attenzione durante le operazioni di backup e restore. Se si ripristina un dump su un server con GTID attivo senza impostare correttamente `--set-gtid-purged`, si possono generare conflitti che rompono la catena di replica.
