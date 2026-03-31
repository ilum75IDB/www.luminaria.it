---
title: "Group Replication"
description: "Mecanismul nativ MySQL pentru replicare sincronă multi-nod cu failover automat și gestionarea quorum-ului."
translationKey: "glossary_group_replication"
aka: "MySQL Group Replication / InnoDB Cluster"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/binary-log-mysql"
---

**Group Replication** este mecanismul nativ al MySQL pentru crearea de clustere cu disponibilitate ridicată și replicare sincronă între mai multe noduri. Spre deosebire de replicarea clasică (asincronă, master-slave), Group Replication garantează că fiecare tranzacție este confirmată de majoritatea nodurilor înainte de a fi considerată committed.

## Cum funcționează

Nodurile comunică printr-un protocol de grup (GCS — Group Communication System) care gestionează consensul distribuit. Fiecare nod menține o copie completă a datelor. Tranzacțiile sunt certificate de grup: dacă nu există conflicte, sunt aplicate pe toate nodurile. Dacă apare un conflict, tranzacția este anulată pe nodul care a inițiat-o.

## Moduri de operare

MySQL Group Replication suportă două moduri: **single-primary** (un singur nod acceptă scrieri, celelalte sunt read-only) și **multi-primary** (toate nodurile acceptă scrieri). Modul single-primary este cel mai folosit în producție pentru că evită conflictele de scriere concurentă.

## De ce contează

Group Replication gestionează automat failover-ul: dacă primary-ul cade, clusterul alege un nou primary dintre secondary în câteva secunde. Acest lucru îl face potrivit pentru medii care necesită disponibilitate ridicată fără intervenție manuală. Necesită minim trei noduri pentru a menține quorum-ul.
