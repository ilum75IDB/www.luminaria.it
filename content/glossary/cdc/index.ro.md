---
title: "CDC"
description: "Change Data Capture — tehnică pentru interceptarea și propagarea modificărilor de date în timp real, adesea bazată pe citirea log-urilor de tranzacții."
translationKey: "glossary_cdc"
aka: "Change Data Capture"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**CDC** (Change Data Capture) este o tehnică pentru interceptarea modificărilor de date (INSERT, UPDATE, DELETE) în momentul în care au loc și propagarea lor către alte sisteme în timp real sau aproape real. Spre deosebire de abordările batch tradiționale (ETL periodic), CDC captează modificările în mod continuu și incremental.

## Cum funcționează

Cea mai răspândită abordare este **log-based CDC**: o componentă externă citește log-urile de tranzacții ale bazei de date (binary log în MySQL, WAL în PostgreSQL, redo log în Oracle) și convertește evenimentele într-un flux de date consumabil de alte sisteme. Instrumente precum Debezium, Maxwell și Canal implementează această abordare pentru MySQL citind direct binary log-urile.

## La ce folosește

CDC este folosit pentru:

- Sincronizarea datelor între baze de date diferite în timp real
- Alimentarea data warehouse-urilor și data lake-urilor cu actualizări incrementale
- Popularea cache-urilor și indexurilor de căutare (Elasticsearch, Redis)
- Implementarea arhitecturilor event-driven și a microserviciilor

## Când se folosește

CDC necesită ca binary log-ul să fie activ și în format ROW (care înregistrează modificările la nivel de rând). Dezactivarea binary log-urilor sau folosirea formatului STATEMENT elimină posibilitatea de a utiliza instrumente CDC, făcând imposibilă integrarea în timp real cu sisteme externe.
