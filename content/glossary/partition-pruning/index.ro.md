---
title: "Partition Pruning"
description: "Mecanism automat Oracle care exclude partițiile nerelevante în timpul execuției unei interogări, citind doar partițiile care conțin date corespunzătoare predicatului."
translationKey: "glossary_partition-pruning"
articles:
  - "/posts/oracle/oracle-partitioning"
  - "/posts/data-warehouse/partitioning-dwh"
---

**Partition Pruning** este mecanismul prin care Oracle, în timpul execuției unei interogări pe o tabelă partițională, identifică și exclude automat partițiile care nu pot conține date relevante pentru predicatul interogării.

## Cum funcționează

Când o interogare include un predicat pe coloana de partiție (ex. `WHERE data_movimento BETWEEN ...`), Oracle consultă metadatele partițiilor și determină care partiții conțin date în intervalul solicitat. Doar acele partiții sunt citite. În planul de execuție apare ca `PARTITION RANGE SINGLE` sau `PARTITION RANGE ITERATOR`.

## La ce folosește

Pe o tabelă de 380 GB cu partiții lunare, o interogare pe o singură lună citește doar ~4 GB în loc de întreaga tabelă. Pruning-ul transformă un full table scan de coșmar într-un full partition scan gestionabil, reducând I/O-ul cu 99%.

## Când se folosește

Pruning-ul este automat, dar funcționează doar cu predicate directe pe coloana de partiție. Aplicarea funcțiilor pe coloană (`TRUNC(data)`, `TO_CHAR(data)`) dezactivează pruning-ul și forțează Oracle să citească toate partițiile. Verifică întotdeauna cu `EXPLAIN PLAN` că pruning-ul este activ.
