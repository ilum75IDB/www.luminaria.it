---
title: "Additive Measure"
description: "Misura numerica in una fact table che può essere sommata lungo tutte le dimensioni — importi, quantità, conteggi. Fondamentale nella progettazione del data warehouse."
translationKey: "glossary_additive_measure"
aka: "Misura additiva, Fully additive measure"
articles:
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

Una **additive measure** (misura additiva) è un valore numerico in una fact table che può essere legittimamente sommato lungo qualsiasi dimensione: per cliente, per prodotto, per periodo, per zona.

## Come funziona

Le misure nelle fact table si classificano in tre categorie:

- **Additive**: possono essere sommate lungo tutte le dimensioni (es. importo vendita, quantità, costo). Sono le più comuni e le più utili
- **Semi-additive**: possono essere sommate lungo alcune dimensioni ma non lungo il tempo (es. saldo di un conto: sommabile per filiale, non per mese)
- **Non-additive**: non possono essere sommate in alcun modo (es. percentuali, rapporti, medie precalcolate)

## A cosa serve

Le misure additive sono il cuore di ogni fact table perché permettono le aggregazioni che il business richiede: totali per periodo, per regione, per prodotto. La regola chiave: memorizzare sempre i valori atomici (il dettaglio), mai gli aggregati. Da un importo per riga di fattura puoi ottenere il totale mensile; dal totale mensile non puoi ricostruire le singole righe.

## Quando si usa

Nella progettazione della fact table, ogni misura va classificata come additiva, semi-additiva o non-additiva. Questo determina quali aggregazioni sono lecite nei report e quali produrrebbero risultati errati. Un errore comune è trattare una misura semi-additiva (come un saldo) come se fosse additiva — sommando saldi mensili per ottenere un "totale" che non ha significato di business.
