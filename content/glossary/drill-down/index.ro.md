---
title: "Drill-down"
description: "Navigare în rapoarte de la un nivel agregat la un nivel de detaliu, tipică analizei OLAP și data warehouse-urilor."
translationKey: "glossary_drill_down"
aka: "Navigare ierarhică, Detaliu progresiv"
articles:
  - "/posts/data-warehouse/ragged-hierarchies"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

**Drill-down-ul** este operațiunea de navigare în rapoarte care permite trecerea de la un nivel agregat la un nivel de detaliu mai mare, coborând de-a lungul unei ierarhii.

## Cum funcționează

Într-o ierarhie Top Group → Group → Client:

1. Se pornește de la nivelul cel mai înalt: cifra de afaceri totală pe Top Group
2. Se face clic pe un Top Group pentru a vedea Group-urile sale (drill-down de primul nivel)
3. Se face clic pe un Group pentru a vedea Clienții individuali (drill-down de al doilea nivel)

Operațiunea inversă — revenirea de la detaliu la agregat — se numește **drill-up** (sau roll-up).

## Cerințe pentru un drill-down corect

Pentru a funcționa fără erori, drill-down-ul necesită:

- O ierarhie **completă**: niciun nivel lipsă (fără NULL-uri)
- **Coerența totalurilor**: suma valorilor la nivel de detaliu trebuie să corespundă totalului de la nivelul superior
- **Structură echilibrată**: toate ramurile ierarhiei trebuie să aibă aceeași adâncime

Dacă ierarhia este dezechilibrată (ragged hierarchy), drill-down-ul produce rezultate incomplete sau eronate. Self-parenting-ul rezolvă aceasta echilibrând structura în amonte.

## Drill-down vs filtru

Drill-down-ul nu este un simplu filtru: este o navigare structurată de-a lungul unei ierarhii predefinite. Un filtru arată un subset de date; un drill-down arată următorul nivel de detaliu într-un context ierarhic.
