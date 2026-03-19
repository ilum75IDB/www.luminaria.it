---
title: "Star schema"
description: "Model de date tipic pentru data warehouse: o fact table centrală conectată la mai multe tabele dimensionale prin chei externe."
translationKey: "glossary_star_schema"
aka: "Schemă stea"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

**Star schema** (schema stea) este cel mai utilizat model de date în data warehouse. Își primește numele de la forma sa: o tabelă centrală de fapte (fact table) conectată la mai multe tabele dimensionale care o înconjoară, ca razele unei stele.

## Structură

- **Fact table** în centru: conține măsurile numerice și cheile externe către dimensiuni
- **Dimension tables** în jurul ei: conțin atributele descriptive (cine, ce, unde, când) cu structură denormalizată

Dimensiunile într-un star schema sunt tipic denormalizate — toate atributele într-o singură tabelă plată, fără ierarhii normalizate. Acest lucru simplifică interogările și îmbunătățește performanța agregărilor.

## De ce funcționează

Star schema este optimizat pentru interogări analitice:

- Join-urile sunt simple: fact table-ul se conectează direct la fiecare dimensiune cu un singur join
- Agregările sunt rapide: optimizatorii bazelor de date recunosc pattern-ul și îl optimizează
- Este intuitiv pentru utilizatorii de business: structura reflectă modul în care aceștia gândesc despre date (vânzări pe produs, pe regiune, pe perioadă)

## Star schema vs Snowflake

**Snowflake schema** normalizează dimensiunile, împărțindu-le în sub-tabele. Economisește spațiu dar complică interogările cu join-uri suplimentare. În practică, star schema este preferat în majoritatea cazurilor deoarece simplitatea interogărilor compensează pe deplin costul spațiului extra din dimensiuni.
