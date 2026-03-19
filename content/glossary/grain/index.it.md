---
title: "Grain"
description: "Il livello di dettaglio di una fact table nel data warehouse — la decisione progettuale che determina quali domande il modello dimensionale può soddisfare."
translationKey: "glossary_grain"
aka: "Granularità, Grana"
articles:
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

Il **grain** (grana, granularità) è il livello di dettaglio di una fact table nel data warehouse. Definisce cosa rappresenta una singola riga: una transazione, un riepilogo giornaliero, un totale mensile, una riga di fattura.

## Come funziona

La scelta del grain è la prima decisione nella progettazione di una fact table. Ogni altra scelta — misure, dimensioni, ETL — discende da essa:

- **Grain fine** (es. riga di fattura): massima flessibilità nelle query, più righe da gestire
- **Grain aggregato** (es. totale mensile per cliente): meno righe, query più veloci, ma impossibilità di scendere nel dettaglio

Il principio fondamentale di Kimball: modellare sempre al livello di dettaglio più fine disponibile nel sistema sorgente.

## A cosa serve

Il grain determina:

- Quali **domande** il data warehouse può soddisfare
- Quali **dimensioni** servono (un grain per riga di fattura richiede dim_prodotto, un grain mensile no)
- Quanto è grande la fact table e quanto dura l'ETL
- Se il **drill-down** nei report è possibile oppure no

## Quando si usa

Il grain si definisce nella fase di progettazione del modello dimensionale, prima di scrivere qualsiasi DDL o ETL. Cambiare il grain dopo il go-live equivale a ricostruire il data warehouse da zero — motivo per cui la scelta iniziale è così critica.
