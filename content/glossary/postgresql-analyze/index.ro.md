---
title: "ANALYZE"
description: "Comanda PostgreSQL care actualizeaza statisticile tabelelor folosite de optimizator pentru a alege planul de executie."
translationKey: "glossary_postgresql_analyze"
aka: "ANALYZE (PostgreSQL)"
articles:
  - "/posts/postgresql/explain-analyze-postgresql"
---

**ANALYZE** este comanda PostgreSQL care colecteaza statistici despre distributia datelor in tabele si le stocheaza in catalogul `pg_statistic` (citibil prin vizualizarea `pg_stats`). Optimizatorul foloseste aceste statistici pentru a estima cardinalitatea — cate randuri va returna fiecare operatie — si a alege cel mai eficient plan de executie.

## Ce colecteaza

Statisticile colectate de ANALYZE includ:

- **Most common values**: valorile cele mai frecvente pentru fiecare coloana si procentul lor
- **Histograme de distributie**: cum sunt distribuite valorile ramase
- **Numarul de valori distincte**: cate valori unice are fiecare coloana
- **Procentul de NULL**: cate randuri au valoarea NULL pentru fiecare coloana

Calitatea acestor statistici depinde de numarul de esantioane colectate, controlat de parametrul `default_statistics_target`.

## De ce conteaza

Fara statistici actualizate, optimizatorul este fortat sa ghiceasca. Estimarile gresite duc la planuri de executie dezastruoase — cum ar fi alegerea unui nested loop pe milioane de randuri crezand ca sunt sute, sau ignorarea unui index perfect adecvat.

## Cand trebuie executat

PostgreSQL executa ANALYZE automat prin autovacuum, dar pragul implicit (50 de randuri + 10% din randurile vii) poate fi prea mare pentru tabelele care cresc rapid. Situatii in care un ANALYZE manual este necesar:

- Dupa importuri masive sau bulk load
- Dupa schimbari semnificative in distributia datelor
- Cand `EXPLAIN ANALYZE` arata estimari de cardinalitate foarte diferite de randurile reale
- Dupa modificarea `default_statistics_target` pentru o coloana
