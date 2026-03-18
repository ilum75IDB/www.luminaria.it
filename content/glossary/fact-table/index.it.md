---
title: "Fact table"
description: "Tabella centrale dello star schema che contiene le misure numeriche e le chiavi esterne verso le tabelle dimensionali."
translationKey: "glossary_fact_table"
aka: "Tabella dei fatti"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

La **fact table** (tabella dei fatti) è la tabella centrale di uno star schema nel data warehouse. Contiene le misure numeriche — importi, quantità, conteggi, durate — e le chiavi esterne che la collegano alle tabelle dimensionali.

## Struttura

Ogni riga della fact table rappresenta un evento o una transazione di business: una vendita, un sinistro, una spedizione, un accesso. Le colonne si dividono in due categorie:

- **Chiavi esterne** (foreign key): puntano alle tabelle dimensionali (chi, cosa, dove, quando)
- **Misure**: i valori numerici da aggregare (importo, quantità, margine)

## Tipi di fact table

- **Transaction fact**: una riga per ogni evento (es. ogni vendita)
- **Periodic snapshot**: una riga per periodo per entità (es. saldo mensile per conto)
- **Accumulating snapshot**: una riga per processo, aggiornata a ogni milestone (es. ciclo ordine-spedizione-fatturazione)

## Relazione con le SCD

Quando le dimensioni usano SCD Tipo 2, la fact table punta alla chiave surrogata della dimensione — non alla chiave naturale. Questo garantisce che ogni fatto sia associato alla versione della dimensione corretta per il momento in cui è accaduto.
