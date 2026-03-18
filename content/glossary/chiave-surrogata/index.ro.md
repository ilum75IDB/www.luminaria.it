---
title: "Cheie surogat"
description: "Identificator numeric generat de data warehouse, distinct de cheia naturală a sistemului sursă. Esențială pentru SCD Tip 2."
translationKey: "glossary_chiave_surrogata"
aka: "Surrogate key"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**Cheia surogat** (surrogate key) este un identificator numeric secvențial generat intern de data warehouse, fără nicio semnificație de business. Este distinctă de cheia naturală — cea care vine din sistemul sursă (ex. codul clientului, numărul de angajat).

## De ce este necesară

În SCD Tip 2, același client poate avea mai multe linii în tabela dimensională — câte una pentru fiecare versiune istorică. Cheia naturală (`client_id`) nu mai este unică, deci este nevoie de un identificator care să distingă fiecare versiune individuală: cheia surogat (`client_key`).

## Cum funcționează

Este generată tipic de o secvență (Oracle) sau o coloană SERIAL/IDENTITY (PostgreSQL, MySQL). Nu este niciodată expusă utilizatorilor finali și nu are semnificație în afara data warehouse-ului.

Tabela de fapte folosește cheia surogat ca cheie externă, indicând spre versiunea specifică a dimensiunii care era validă în momentul faptului. Acest lucru garantează că fiecare tranzacție este asociată cu contextul dimensional corect pentru acel moment în timp.

## Avantaje

- Permite versionarea dimensiunilor (SCD Tip 2)
- Join-urile între fapte și dimensiuni sunt pe numere întregi, deci rapide
- Izolează DWH-ul de schimbările în cheile sistemelor sursă
- Suportă încărcarea din surse multiple cu chei naturale potențial duplicate
