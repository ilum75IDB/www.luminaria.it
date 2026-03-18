---
title: "Data Warehouse"
description: "Sistema centralizzato di raccolta e storicizzazione dati provenienti da fonti diverse, progettato per l'analisi e il supporto alle decisioni aziendali."
translationKey: "glossary_data-warehouse"
articles:
  - "/posts/project-management/4-milioni-nessun-software"
---

Un **Data Warehouse** (DWH) è un sistema di archiviazione dati progettato specificamente per l'analisi, il reporting e il supporto alle decisioni aziendali. A differenza dei database operazionali (OLTP), un DWH raccoglie dati da molteplici fonti, li trasforma e li organizza in strutture ottimizzate per le interrogazioni analitiche.

## Come funziona

I dati vengono estratti dai sistemi sorgente (gestionali, CRM, ERP), trasformati attraverso processi ETL che li puliscono, normalizzano e arricchiscono, e infine caricati nel DWH. Il modello dati tipico è lo star schema: una fact table centrale con le misure numeriche collegata a tabelle dimensionali che descrivono il contesto (tempo, cliente, prodotto, geografia).

## A cosa serve

Un DWH permette di rispondere a domande di business che i sistemi operazionali non possono gestire: trend storici, analisi comparative tra periodi, aggregazioni cross-system, KPI aziendali. Separa il carico analitico da quello transazionale, evitando che le query di reporting impattino le performance delle applicazioni operative.

## Quando si usa

Un DWH è necessario quando un'azienda ha bisogno di integrare dati da fonti diverse per produrre analisi consolidate. La complessità e i costi dipendono dal numero di sistemi sorgente, dal volume dei dati e dalla frequenza di aggiornamento richiesta.
