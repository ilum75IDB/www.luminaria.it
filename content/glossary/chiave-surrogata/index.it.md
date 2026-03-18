---
title: "Chiave surrogata"
description: "Identificativo numerico generato dal data warehouse, distinto dalla chiave naturale del sistema sorgente. Indispensabile nella SCD Tipo 2."
translationKey: "glossary_chiave_surrogata"
aka: "Surrogate key"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

La **chiave surrogata** (surrogate key) è un identificativo numerico sequenziale generato internamente dal data warehouse, senza alcun significato di business. È distinta dalla chiave naturale — quella che arriva dal sistema sorgente (es. il codice cliente, il numero di matricola).

## Perché serve

Nella SCD Tipo 2, lo stesso cliente può avere più righe nella tabella dimensionale — una per ogni versione storica. La chiave naturale (`cliente_id`) non è più univoca, quindi serve un identificativo che distingua ogni singola versione: la chiave surrogata (`cliente_key`).

## Come funziona

Viene generata tipicamente da una sequence (Oracle) o una colonna SERIAL/IDENTITY (PostgreSQL, MySQL). Non viene mai esposta agli utenti finali e non ha significato al di fuori del data warehouse.

La fact table usa la chiave surrogata come foreign key, puntando alla versione specifica della dimensione valida al momento del fatto. Questo garantisce che ogni transazione sia associata al contesto dimensionale corretto per quel momento nel tempo.

## Vantaggi

- Permette il versioning delle dimensioni (SCD Tipo 2)
- I join tra fact e dimension sono su interi, quindi veloci
- Isola il DWH dai cambiamenti nelle chiavi dei sistemi sorgente
- Supporta il caricamento da fonti multiple con chiavi naturali potenzialmente duplicate
