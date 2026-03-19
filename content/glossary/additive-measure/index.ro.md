---
title: "Additive Measure"
description: "Măsură numerică într-o fact table care poate fi sumată de-a lungul tuturor dimensiunilor — sume, cantități, contorizări. Fundamentală în proiectarea data warehouse-ului."
translationKey: "glossary_additive_measure"
aka: "Măsură aditivă, Fully additive measure"
articles:
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

O **additive measure** (măsură aditivă) este o valoare numerică într-o fact table care poate fi sumată legitim de-a lungul oricărei dimensiuni: pe client, pe produs, pe perioadă, pe zonă.

## Cum funcționează

Măsurile din fact table se clasifică în trei categorii:

- **Aditive**: pot fi sumate de-a lungul tuturor dimensiunilor (ex. sumă vânzare, cantitate, cost). Cele mai comune și cele mai utile
- **Semi-aditive**: pot fi sumate de-a lungul unor dimensiuni dar nu de-a lungul timpului (ex. sold al unui cont: sumabil pe sucursală, nu pe lună)
- **Non-aditive**: nu pot fi sumate deloc (ex. procente, rapoarte, medii precalculate)

## La ce servește

Măsurile aditive sunt inima fiecărei fact table deoarece permit agregările pe care business-ul le cere: totaluri pe perioadă, pe regiune, pe produs. Regula cheie: stochează întotdeauna valorile atomice (detaliul), niciodată agregatele. Dintr-o sumă pe linie de factură poți obține totalul lunar; din totalul lunar nu poți reconstitui liniile individuale.

## Când se folosește

La proiectarea fact table-ului, fiecare măsură trebuie clasificată ca aditivă, semi-aditivă sau non-aditivă. Aceasta determină ce agregări sunt valide în rapoarte și care ar produce rezultate incorecte. O greșeală frecventă este tratarea unei măsuri semi-aditive (precum un sold) ca și cum ar fi aditivă — sumând solduri lunare pentru a obține un „total" care nu are semnificație de business.
