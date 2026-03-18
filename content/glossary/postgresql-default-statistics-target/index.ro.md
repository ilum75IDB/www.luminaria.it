---
title: "default_statistics_target"
description: "Parametrul PostgreSQL care controleaza cate esantioane colecteaza ANALYZE pentru a estima distributia datelor in fiecare coloana."
translationKey: "glossary_postgresql_default_statistics_target"
aka: "default_statistics_target (PostgreSQL)"
articles:
  - "/posts/postgresql/explain-analyze-postgresql"
---

**default_statistics_target** este parametrul PostgreSQL care defineste numarul de esantioane colectate de comanda `ANALYZE` pentru a construi statisticile fiecarei coloane. Valoarea implicita este 100.

## Cum functioneaza

PostgreSQL esantioneaza un anumit numar de valori pentru fiecare coloana si le foloseste pentru a construi doua structuri:

- **Most common values (MCV)**: lista valorilor celor mai frecvente, cu frecventele respective
- **Histograma**: distributia valorilor ramase, impartita in bucket-uri de populatie egala

Parametrul `default_statistics_target` determina cate elemente vor avea aceste structuri. Cu valoarea implicita de 100, histograma va avea 100 de bucket-uri si lista MCV va contine pana la 100 de valori.

## Cand trebuie crescut

Pentru tabele mici sau cu distributie uniforma, 100 de esantioane sunt suficiente. Pentru tabele mari cu distributie asimetrica (skewed) — unde putine valori domina majoritatea randurilor — 100 de esantioane pot da o reprezentare distorsionata, ducand optimizatorul la estimari de cardinalitate gresite.

Se poate creste target-ul la nivel de coloana individuala:

    ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
    ANALYZE orders;

Valori intre 500 si 1000 imbunatatesc semnificativ calitatea estimarilor pe coloane cu distributie neuniforma.

## Limite practice

Peste 1000 beneficiul este marginal si `ANALYZE`-ul insusi devine mai lent, pentru ca trebuie sa esantioneze mai multe randuri si sa construiasca structuri mai mari. Este o reglare fina: trebuie aplicata doar coloanelor care cauzeaza efectiv estimari gresite, nu tuturor coloanelor din toate tabelele.
