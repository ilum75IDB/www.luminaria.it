---
title: "Partition Pruning"
description: "Meccanismo automatico di Oracle che esclude le partizioni non rilevanti durante l'esecuzione di una query, leggendo solo le partizioni che contengono dati corrispondenti al predicato."
translationKey: "glossary_partition-pruning"
articles:
  - "/posts/oracle/oracle-partitioning"
  - "/posts/data-warehouse/partitioning-dwh"
---

Il **Partition Pruning** è il meccanismo con cui Oracle, durante l'esecuzione di una query su una tabella partizionata, identifica ed esclude automaticamente le partizioni che non possono contenere dati rilevanti per il predicato della query.

## Come funziona

Quando una query include un predicato sulla colonna di partizione (es. `WHERE data_movimento BETWEEN ...`), Oracle consulta i metadati delle partizioni e determina quali partizioni contengono dati nell'intervallo richiesto. Solo quelle partizioni vengono lette. Nel piano di esecuzione appare come `PARTITION RANGE SINGLE` o `PARTITION RANGE ITERATOR`.

## A cosa serve

Su una tabella da 380 GB con partizioni mensili, una query su un singolo mese legge solo ~4 GB invece dell'intera tabella. Il pruning trasforma un full table scan da incubo in un full partition scan gestibile, riducendo I/O del 99%.

## Quando si usa

Il pruning è automatico, ma funziona solo con predicati diretti sulla colonna di partizione. Applicare funzioni alla colonna (`TRUNC(data)`, `TO_CHAR(data)`) disabilita il pruning e forza Oracle a leggere tutte le partizioni. Verificare sempre con `EXPLAIN PLAN` che il pruning sia attivo.
