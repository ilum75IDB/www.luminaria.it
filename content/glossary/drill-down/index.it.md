---
title: "Drill-down"
description: "Navigazione nei report dal livello aggregato al livello di dettaglio, tipica dell'analisi OLAP e dei data warehouse."
translationKey: "glossary_drill_down"
aka: "Navigazione gerarchica, Dettaglio progressivo"
articles:
  - "/posts/data-warehouse/ragged-hierarchies"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

Il **drill-down** è l'operazione di navigazione nei report che permette di passare da un livello aggregato a un livello di maggiore dettaglio, scendendo lungo una gerarchia.

## Come funziona

In una gerarchia Top Group → Group → Client:

1. Si parte dal livello più alto: fatturato totale per Top Group
2. Si clicca su un Top Group per vedere i suoi Group (drill-down di primo livello)
3. Si clicca su un Group per vedere i singoli Client (drill-down di secondo livello)

L'operazione inversa — risalire dal dettaglio all'aggregato — si chiama **drill-up** (o roll-up).

## Requisiti per un drill-down corretto

Per funzionare senza errori, il drill-down richiede:

- Una gerarchia **completa**: nessun livello mancante (niente NULL)
- **Coerenza dei totali**: la somma dei valori al livello di dettaglio deve corrispondere al totale del livello superiore
- **Struttura bilanciata**: tutti i rami della gerarchia devono avere la stessa profondità

Se la gerarchia è sbilanciata (ragged hierarchy), il drill-down produce risultati incompleti o errati. Il self-parenting risolve questo problema bilanciando la struttura a monte.

## Drill-down vs filtro

Il drill-down non è un semplice filtro: è una navigazione strutturata lungo una gerarchia predefinita. Un filtro mostra un sottoinsieme dei dati; un drill-down mostra il livello successivo di dettaglio all'interno di un contesto gerarchico.
