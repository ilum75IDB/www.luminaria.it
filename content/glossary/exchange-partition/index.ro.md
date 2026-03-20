---
title: "Exchange Partition"
description: "Operațiune DDL Oracle care schimbă instantaneu segmentele de date între o tabelă nepartiționată și o partiție, fără a muta fizic niciun dat."
translationKey: "glossary_exchange-partition"
aka: "Schimb de partiție"
articles:
  - "/posts/data-warehouse/partitioning-dwh"
---

**Exchange Partition** este o operațiune DDL Oracle care permite schimbarea instantanee a conținutului unei partiții cu cel al unei tabele nepartiționate. Nu se mută niciun byte de date — operațiunea modifică doar pointerii din data dictionary.

## Cum funcționează

Comanda `ALTER TABLE ... EXCHANGE PARTITION ... WITH TABLE ...` modifică metadatele din data dictionary astfel încât segmentele fizice ale partiției și ale tabelei de staging să își schimbe proprietarul. Tabela de staging devine partiția și invers. Operațiunea durează mai puțin de o secundă indiferent de volumul datelor, deoarece nu implică nicio mișcare fizică.

## La ce servește

În data warehouse-uri, exchange partition este instrumentul principal pentru încărcarea masivă a datelor. Procesul tipic este: ETL-ul încarcă datele într-o tabelă de staging, construiește indexurile, validează datele, și apoi execută exchange-ul cu partiția țintă. În timpul exchange-ului, query-urile pe celelalte partiții continuă să funcționeze fără întrerupere.

## Ce poate merge prost

Clauza `WITHOUT VALIDATION` omite verificarea că datele din tabela de staging se încadrează efectiv în intervalul partiției — accelerează operațiunea dar necesită ca ETL-ul să garanteze corectitudinea datelor. Dacă datele din staging conțin date în afara intervalului, ajung în partiția greșită fără nicio eroare. Clauza `INCLUDING INDEXES` necesită ca tabela de staging să aibă indexuri cu aceeași structură ca indexurile locale ale tabelei partiționate.
