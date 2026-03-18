---
title: "CDC"
description: "Change Data Capture — tecnica per intercettare e propagare le modifiche ai dati in tempo reale, spesso basata sulla lettura dei log delle transazioni."
translationKey: "glossary_cdc"
aka: "Change Data Capture"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**CDC** (Change Data Capture) è una tecnica per intercettare le modifiche ai dati (INSERT, UPDATE, DELETE) nel momento in cui avvengono e propagarle verso altri sistemi in tempo reale o quasi reale. A differenza degli approcci batch tradizionali (ETL periodici), il CDC cattura i cambiamenti in modo continuo e incrementale.

## Come funziona

L'approccio più diffuso è il **log-based CDC**: un componente esterno legge i log delle transazioni del database (binary log in MySQL, WAL in PostgreSQL, redo log in Oracle) e converte gli eventi in un flusso di dati consumabile da altri sistemi. Strumenti come Debezium, Maxwell e Canal implementano questo approccio per MySQL leggendo direttamente i binary log.

## A cosa serve

Il CDC è usato per:

- Sincronizzare dati tra database diversi in tempo reale
- Alimentare data warehouse e data lake con aggiornamenti incrementali
- Popolare cache e indici di ricerca (Elasticsearch, Redis)
- Implementare architetture event-driven e microservizi

## Quando si usa

Il CDC richiede che il binary log sia attivo e in formato ROW (che registra le modifiche riga per riga). Disabilitare i binary log o usare il formato STATEMENT elimina la possibilità di utilizzare strumenti di CDC, rendendo impossibile l'integrazione in tempo reale con sistemi esterni.
