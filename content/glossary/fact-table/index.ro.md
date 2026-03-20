---
title: "Fact table"
description: "Tabela centrală a star schema-ului care conține măsurile numerice și cheile externe către tabelele dimensionale."
translationKey: "glossary_fact_table"
aka: "Tabelă de fapte"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
  - "/posts/data-warehouse/partitioning-dwh"
---

**Fact table** (tabela de fapte) este tabela centrală a unui star schema în data warehouse. Conține măsurile numerice — sume, cantități, contorizări, durate — și cheile externe care o conectează la tabelele dimensionale.

## Structură

Fiecare linie din fact table reprezintă un eveniment sau o tranzacție de business: o vânzare, o daună, o expediere, un acces. Coloanele se împart în două categorii:

- **Chei externe** (foreign keys): indică spre tabelele dimensionale (cine, ce, unde, când)
- **Măsuri**: valorile numerice de agregat (sumă, cantitate, marjă)

## Tipuri de fact tables

- **Transaction fact**: o linie per eveniment (ex. fiecare vânzare)
- **Periodic snapshot**: o linie per perioadă per entitate (ex. sold lunar per cont)
- **Accumulating snapshot**: o linie per proces, actualizată la fiecare milestone (ex. ciclul comandă-expediere-facturare)

## Relația cu SCD

Când dimensiunile folosesc SCD Tip 2, fact table-ul indică spre cheia surogat a dimensiunii — nu spre cheia naturală. Acest lucru garantează că fiecare fapt este asociat cu versiunea corectă a dimensiunii pentru momentul în care s-a produs.
