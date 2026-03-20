---
title: "Range Partitioning"
description: "Strategie de partiționare care împarte o tabelă în segmente bazate pe intervale de valori ale unei coloane, de obicei o dată."
translationKey: "glossary_range-partitioning"
aka: "Partiționare pe intervale"
articles:
  - "/posts/data-warehouse/partitioning-dwh"
  - "/posts/oracle/oracle-partitioning"
---

**Range Partitioning** (partiționare pe intervale) este o strategie de partiționare a tabelelor în care rândurile sunt distribuite în partiții diferite pe baza valorii unei coloane relative la intervale predefinite. Coloana de partiționare este aproape întotdeauna o dată în data warehouse-uri.

## Cum funcționează

Fiecare partiție este definită cu o clauză `VALUES LESS THAN` care stabilește limita superioară a intervalului. Oracle atribuie automat fiecare rând partiției corecte pe baza valorii coloanei de partiționare. Dacă un rând are `data_vendita = '2025-03-15'`, este inserat în partiția al cărei interval include acea dată.

## Când se folosește

Range partitioning este alegerea naturală când datele au o dimensiune temporală dominantă — fact table-uri în data warehouse-uri, tabele de log, tabele de tranzacții. Granularitatea partiției (zilnică, lunară, trimestrială) depinde de volumul de inserare și de tipologia query-urilor: partițiile prea mici generează overhead de gestionare, cele prea mari reduc eficiența partition pruning-ului.

## Avantaje operaționale

Dincolo de performanța query-urilor, range partitioning permite operațiuni de gestionare a ciclului de viață al datelor imposibile pe tabele monolitice: drop instantaneu al unei partiții (fără DELETE), compresie selectivă a partițiilor istorice, mutare pe storage diferit (ILM — Information Lifecycle Management), și exchange partition pentru încărcări masive cu impact zero.
