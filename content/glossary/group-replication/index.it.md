---
title: "Group Replication"
description: "Meccanismo nativo di MySQL per la replica sincrona multi-nodo con failover automatico e gestione del quorum."
translationKey: "glossary_group_replication"
aka: "MySQL Group Replication / InnoDB Cluster"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/binary-log-mysql"
---

**Group Replication** è il meccanismo nativo di MySQL per creare cluster ad alta disponibilità con replica sincrona tra più nodi. A differenza della replica classica (asincrona, master-slave), Group Replication garantisce che ogni transazione sia confermata dalla maggioranza dei nodi prima di essere considerata committata.

## Come funziona

I nodi comunicano tramite un protocollo di gruppo (GCS — Group Communication System) che gestisce il consenso distribuito. Ogni nodo mantiene una copia completa dei dati. Le transazioni vengono certificate dal gruppo: se non ci sono conflitti, vengono applicate su tutti i nodi. Se c'è un conflitto, la transazione viene annullata sul nodo che l'ha originata.

## Modalità operative

MySQL Group Replication supporta due modalità: **single-primary** (un solo nodo accetta scritture, gli altri sono in sola lettura) e **multi-primary** (tutti i nodi accettano scritture). La modalità single-primary è la più usata in produzione perché evita i conflitti di scrittura concorrente.

## Perché è critico

Group Replication gestisce automaticamente il failover: se il primary cade, il cluster elegge un nuovo primary tra i secondary in pochi secondi. Questo lo rende adatto per ambienti che richiedono alta disponibilità senza intervento manuale. Richiede un minimo di tre nodi per mantenere il quorum.
