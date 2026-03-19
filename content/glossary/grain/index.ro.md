---
title: "Grain"
description: "Nivelul de detaliu al unei fact table într-un data warehouse — decizia de proiectare care determină la ce întrebări poate răspunde modelul dimensional."
translationKey: "glossary_grain"
aka: "Granularitate, Nivel de detaliu"
articles:
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

**Grain-ul** (granularitatea) este nivelul de detaliu al unei fact table într-un data warehouse. Definește ce reprezintă un singur rând: o tranzacție, un sumar zilnic, un total lunar, o linie de factură.

## Cum funcționează

Alegerea grain-ului este prima decizie la proiectarea unei fact table. Toate celelalte decizii — măsuri, dimensiuni, ETL — decurg din ea:

- **Grain fin** (ex. linie de factură): flexibilitate maximă în interogări, mai multe rânduri de gestionat
- **Grain agregat** (ex. total lunar per client): mai puține rânduri, interogări mai rapide, dar imposibilitatea de a coborî în detaliu

Principiul fundamental al lui Kimball: modelează întotdeauna la cel mai fin nivel de detaliu disponibil în sistemul sursă.

## La ce servește

Grain-ul determină:

- La ce **întrebări** poate răspunde data warehouse-ul
- Ce **dimensiuni** sunt necesare (un grain la nivel de linie necesită dim_produs, un grain lunar nu)
- Cât de mare este fact table-ul și cât durează ETL-ul
- Dacă **drill-down-ul** în rapoarte este posibil sau nu

## Când se folosește

Grain-ul se definește în faza de proiectare a modelului dimensional, înainte de a scrie orice DDL sau ETL. Schimbarea grain-ului după go-live echivalează cu reconstruirea data warehouse-ului de la zero — motiv pentru care alegerea inițială este atât de critică.
