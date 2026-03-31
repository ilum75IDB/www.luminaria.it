---
title: "Single-primary"
description: "Modul MySQL Group Replication în care un singur nod acceptă scrieri, iar celelalte sunt read-only cu failover automat."
translationKey: "glossary_single_primary"
aka: "Single-primary mode"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
---

**Single-primary** este modul de operare cel mai frecvent al MySQL Group Replication, în care un singur nod din cluster — primary-ul — acceptă operații de scriere. Celelalte noduri (secondary) sunt read-only (`read_only=ON`, `super_read_only=ON`) și primesc modificările prin replicarea sincronă a grupului.

## Cum funcționează

Parametrul `group_replication_single_primary_mode=ON` activează acest mod. Primary-ul este singurul nod cu `read_only=OFF`. Dacă primary-ul este oprit sau devine inaccesibil, clusterul declanșează o alegere automată și unul dintre secondary devine noul primary în câteva secunde.

## De ce se folosește

Modul single-primary evită conflictele de scriere concurentă tipice pentru multi-primary. În producție, majoritatea clusterelor MySQL folosesc acest mod pentru că este mai previzibil: aplicațiile scriu pe un singur endpoint, replicarea este liniară și debugging-ul este mai simplu.

## Ce poate merge prost

Când primary-ul este oprit pentru mentenanță, clusterul face un failover automat. În acele secunde, conexiunile active pot fi întrerupte și tranzacțiile în curs pot eșua. Este o întrerupere scurtă dar trebuie comunicată. Regula practică: într-o intervenție de mentenanță pe un cluster single-primary, secondary-urile se ating primele, primary-ul la urmă.
