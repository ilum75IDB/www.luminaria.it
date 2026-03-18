---
title: "default_statistics_target"
description: "Il parametro PostgreSQL che controlla quanti campioni raccoglie ANALYZE per stimare la distribuzione dei dati in ogni colonna."
translationKey: "glossary_postgresql_default_statistics_target"
aka: "default_statistics_target (PostgreSQL)"
articles:
  - "/posts/postgresql/explain-analyze-postgresql"
---

**default_statistics_target** è il parametro PostgreSQL che definisce il numero di campioni raccolti dal comando `ANALYZE` per costruire le statistiche di ogni colonna. Il valore di default è 100.

## Come funziona

PostgreSQL campiona un certo numero di valori per ogni colonna e li usa per costruire due strutture:

- **Most common values (MCV)**: la lista dei valori più frequenti, con le rispettive frequenze
- **Istogramma**: la distribuzione dei valori rimanenti, divisa in bucket di uguale popolazione

Il parametro `default_statistics_target` determina quanti elementi avranno queste strutture. Con il valore 100 (default), l'istogramma avrà 100 bucket e la lista MCV conterrà fino a 100 valori.

## Quando aumentarlo

Per tabelle piccole o con distribuzione uniforme, 100 campioni sono sufficienti. Per tabelle grandi con distribuzione asimmetrica (skewed) — dove pochi valori dominano la maggior parte delle righe — 100 campioni possono dare una rappresentazione distorta, portando l'optimizer a stime di cardinalità sbagliate.

Si può aumentare il target a livello di singola colonna:

    ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
    ANALYZE orders;

Valori tra 500 e 1000 migliorano sensibilmente la qualità delle stime su colonne con distribuzione non uniforme.

## Limiti pratici

Oltre 1000 il beneficio è marginale e l'`ANALYZE` stesso diventa più lento, perché deve campionare più righe e costruire strutture più grandi. È una regolazione fine: va applicata solo alle colonne che effettivamente causano stime errate, non a tutte le colonne di tutte le tabelle.
