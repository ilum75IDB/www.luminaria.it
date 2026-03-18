---
title: "Kimball"
description: "Ralph Kimball — metodologie de proiectare a data warehouse-ului bazată pe dimensional modeling, star schema și procese ETL bottom-up."
translationKey: "glossary_kimball"
aka: "Metodologia Kimball, Dimensional Modeling"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**Kimball** se referă la Ralph Kimball și la metodologia sa de proiectare a data warehouse-urilor, descrisă în cartea *The Data Warehouse Toolkit* (prima ediție 1996, a treia ediție 2013).

## Abordarea

Metodologia Kimball se bazează pe trei piloni:

- **Dimensional modeling**: organizarea datelor în star schema cu fact tables și dimension tables, optimizate pentru interogări analitice
- **Bottom-up**: construirea DWH-ului pornind de la data mart-uri departamentale individuale, integrându-le progresiv prin dimensiuni conforme (conformed dimensions)
- **Bus architecture**: un framework pentru a garanta coerența între data mart-uri prin dimensiuni și fapte partajate

## Slowly Changing Dimensions

Kimball a definit clasificarea SCD (Slowly Changing Dimensions) în tipurile de la 0 la 7, care a devenit standardul de facto în industrie. Tipul 2 — cu chei surogat și date de valabilitate — este cel mai utilizat pentru urmărirea istoriei dimensiunilor.

## Kimball vs Inmon

Alternativa principală este metodologia lui Bill Inmon, care propune o abordare top-down cu un enterprise data warehouse normalizat (3NF) din care se derivă data mart-urile. Cele două metodologii nu sunt mutual exclusive și multe proiecte reale adoptă elemente din ambele.
