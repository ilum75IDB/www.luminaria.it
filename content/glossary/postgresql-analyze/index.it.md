---
title: "ANALYZE"
description: "Il comando PostgreSQL che aggiorna le statistiche delle tabelle usate dall'optimizer per scegliere il piano di esecuzione."
translationKey: "glossary_postgresql_analyze"
aka: "ANALYZE (PostgreSQL)"
articles:
  - "/posts/postgresql/explain-analyze-postgresql"
---

**ANALYZE** è il comando PostgreSQL che raccoglie statistiche sulla distribuzione dei dati nelle tabelle e le salva nel catalogo `pg_statistic` (leggibile tramite la vista `pg_stats`). L'optimizer usa queste statistiche per stimare la cardinalità — quante righe restituirà ogni operazione — e scegliere il piano di esecuzione più efficiente.

## Cosa raccoglie

Le statistiche raccolte da ANALYZE includono:

- **Most common values**: i valori più frequenti per ogni colonna e la loro percentuale
- **Istogrammi di distribuzione**: come sono distribuiti i valori rimanenti
- **Numero di valori distinti**: quanti valori unici ha ogni colonna
- **Percentuale di NULL**: quante righe hanno valore NULL per ogni colonna

La qualità di queste statistiche dipende dal numero di campioni raccolti, controllato dal parametro `default_statistics_target`.

## Perché è critico

Senza statistiche aggiornate, l'optimizer è costretto a indovinare. Le stime sbagliate portano a piani di esecuzione disastrosi — come scegliere un nested loop su milioni di righe pensando che siano centinaia, o ignorare un indice perfettamente adatto.

## Quando eseguirlo

PostgreSQL esegue ANALYZE automaticamente tramite l'autovacuum, ma la soglia di default (50 righe + 10% delle righe vive) può essere troppo alta per tabelle che crescono rapidamente. Situazioni in cui serve un ANALYZE manuale:

- Dopo import massivi o bulk load
- Dopo cambiamenti significativi nella distribuzione dei dati
- Quando `EXPLAIN ANALYZE` mostra stime di cardinalità molto diverse dalle righe reali
- Dopo aver modificato il `default_statistics_target` di una colonna
