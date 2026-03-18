---
title: "Kimball"
description: "Ralph Kimball — metodologia di progettazione data warehouse basata su dimensional modeling, star schema e processi ETL bottom-up."
translationKey: "glossary_kimball"
aka: "Metodologia Kimball, Dimensional Modeling"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**Kimball** si riferisce a Ralph Kimball e alla sua metodologia di progettazione dei data warehouse, descritta nel libro *The Data Warehouse Toolkit* (prima edizione 1996, terza edizione 2013).

## L'approccio

La metodologia Kimball si basa su tre pilastri:

- **Dimensional modeling**: organizzare i dati in star schema con fact table e dimension table, ottimizzati per le query analitiche
- **Bottom-up**: costruire il DWH partendo dai singoli data mart dipartimentali, integrandoli progressivamente tramite dimensioni conformi (conformed dimensions)
- **Bus architecture**: un framework per garantire coerenza tra i data mart attraverso dimensioni e fatti condivisi

## Le Slowly Changing Dimensions

Kimball ha definito la classificazione delle SCD (Slowly Changing Dimensions) nei tipi da 0 a 7, che è diventata lo standard de facto nel settore. Il Tipo 2 — con chiavi surrogate e date di validità — è il più usato per tracciare la storia delle dimensioni.

## Kimball vs Inmon

L'alternativa principale è la metodologia di Bill Inmon, che propone un approccio top-down con un enterprise data warehouse normalizzato (3NF) da cui derivano i data mart. Le due metodologie non sono mutualmente esclusive e molti progetti reali adottano elementi di entrambe.
