---
title: "Star schema"
description: "Modello di dati tipico del data warehouse: una fact table al centro collegata a più tabelle dimensionali tramite chiavi esterne."
translationKey: "glossary_star_schema"
aka: "Schema a stella"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

Lo **star schema** (schema a stella) è il modello di dati più usato nel data warehouse. Prende il nome dalla sua forma: una tabella centrale dei fatti (fact table) collegata a più tabelle dimensionali che le stanno intorno, come i raggi di una stella.

## Struttura

- **Fact table** al centro: contiene le misure numeriche e le foreign key verso le dimensioni
- **Dimension tables** intorno: contengono gli attributi descrittivi (chi, cosa, dove, quando) con struttura denormalizzata

Le dimensioni nello star schema sono tipicamente denormalizzate — tutti gli attributi in una sola tabella piatta, senza gerarchie normalizzate. Questo semplifica le query e migliora le performance delle aggregazioni.

## Perché funziona

Lo star schema è ottimizzato per le query analitiche:

- I join sono semplici: la fact si collega direttamente a ogni dimensione con un singolo join
- Le aggregazioni sono veloci: gli optimizer dei database riconoscono il pattern e lo ottimizzano
- È intuitivo per gli utenti business: la struttura rispecchia il modo in cui pensano ai dati (vendite per prodotto, per regione, per periodo)

## Star schema vs Snowflake

Lo **snowflake schema** normalizza le dimensioni, spezzandole in sotto-tabelle. Risparmia spazio ma complica le query con join aggiuntivi. Nella pratica, lo star schema è preferito nella maggior parte dei casi perché la semplicità delle query compensa ampiamente il costo dello spazio extra nelle dimensioni.
