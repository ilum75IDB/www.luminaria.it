---
title: "WSREP"
description: "Write Set Replication — API e protocollo di replica sincrona usato da Galera Cluster per mantenere i nodi del cluster allineati in tempo reale."
translationKey: "glossary_wsrep"
aka: "Write Set Replication"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**WSREP** (Write Set Replication) è l'API e il protocollo che Galera Cluster utilizza per la replica sincrona multi-master. Ogni transazione viene catturata come "write set" (insieme di modifiche a livello di riga) e replicata su tutti i nodi del cluster prima del commit.

## Come funziona

Quando un nodo esegue una transazione, WSREP la intercetta al momento del commit, la impacchetta come write set e la invia a tutti i nodi del cluster tramite il protocollo di comunicazione di gruppo. Ogni nodo esegue un processo di **certification**: verifica che la transazione non sia in conflitto con altre transazioni concorrenti. Se la certification ha successo, tutti i nodi applicano la transazione. Se fallisce, la transazione viene annullata sul nodo che l'ha originata.

## A cosa serve

WSREP garantisce che tutti i nodi del cluster abbiano gli stessi dati in ogni momento (replica sincrona). A differenza della replica asincrona tradizionale di MySQL, non c'è ritardo tra master e slave: quando una transazione è committata su un nodo, è già presente su tutti gli altri.

## Quando si usa

WSREP si attiva con il parametro `wsrep_on=ON` nella configurazione di MariaDB/Percona XtraDB Cluster. Le variabili di stato che iniziano con `wsrep_` (come `wsrep_cluster_size`, `wsrep_cluster_status`, `wsrep_flow_control_paused`) sono gli indicatori principali per monitorare la salute del cluster.
