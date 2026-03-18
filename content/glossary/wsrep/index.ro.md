---
title: "WSREP"
description: "Write Set Replication — API și protocol de replicare sincronă folosit de Galera Cluster pentru menținerea nodurilor clusterului aliniate în timp real."
translationKey: "glossary_wsrep"
aka: "Write Set Replication"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**WSREP** (Write Set Replication) este API-ul și protocolul pe care Galera Cluster îl folosește pentru replicarea sincronă multi-master. Fiecare tranzacție este capturată ca "write set" (set de modificări la nivel de rând) și replicată pe toate nodurile clusterului înainte de commit.

## Cum funcționează

Când un nod execută o tranzacție, WSREP o interceptează la momentul commit-ului, o împachetează ca write set și o trimite tuturor nodurilor clusterului prin protocolul de comunicare de grup. Fiecare nod execută un proces de **certification**: verifică dacă tranzacția nu intră în conflict cu alte tranzacții concurente. Dacă certification-ul are succes, toate nodurile aplică tranzacția. Dacă eșuează, tranzacția este anulată pe nodul care a inițiat-o.

## La ce folosește

WSREP garantează că toate nodurile clusterului au aceleași date în orice moment (replicare sincronă). Spre deosebire de replicarea asincronă tradițională MySQL, nu există întârziere între master și slave: când o tranzacție este confirmată pe un nod, este deja prezentă pe toate celelalte.

## Când se folosește

WSREP se activează cu parametrul `wsrep_on=ON` în configurarea MariaDB/Percona XtraDB Cluster. Variabilele de stare care încep cu `wsrep_` (precum `wsrep_cluster_size`, `wsrep_cluster_status`, `wsrep_flow_control_paused`) sunt indicatorii principali pentru monitorizarea sănătății clusterului.
