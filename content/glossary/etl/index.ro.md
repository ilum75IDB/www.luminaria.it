---
title: "ETL"
description: "Extract, Transform, Load — procesul de extragere, transformare si incarcare a datelor din sistemele sursa in data warehouse."
translationKey: "glossary_etl"
aka: "Extract, Transform, Load"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
  - "/posts/data-warehouse/ragged-hierarchies"
  - "/posts/data-warehouse/fatto-grana-sbagliata"
---

**ETL** (Extract, Transform, Load) este procesul fundamental prin care datele sunt mutate din sistemele sursa (baze de date operationale, fisiere, API-uri) in data warehouse.

## Cele trei faze

- **Extract**: extragerea datelor din sistemele sursa. Poate fi completa (full load) sau incrementala (doar date noi sau modificate)
- **Transform**: curatarea, validarea, standardizarea si imbogatirea datelor. Aici se aplica regulile de business, lookup-urile pe dimensiuni, calculele derivate
- **Load**: incarcarea datelor transformate in tabelele data warehouse-ului (fact si dimension)

## De ce este critic

ETL este partea cea mai putin vizibila dar cea mai critica a unui data warehouse. Daca datele sunt extrase incomplet, transformate cu reguli eronate sau incarcate fara verificari, tot ce sta deasupra — rapoarte, dashboard-uri, decizii — va fi gresit.

Un ETL bine proiectat determina si fereastra de incarcare: cat timp este necesar pentru actualizarea data warehouse-ului. In medii reale, trecerea de la 4 ore la 25 de minute poate face diferenta intre date actualizate dimineata sau dupa-amiaza.

## ELT vs ETL

Odata cu aparitia data warehouse-urilor cloud si a motoarelor columnare de inalta performanta, s-a raspandit pattern-ul **ELT** (Extract, Load, Transform): datele sunt incarcate brute in warehouse si transformate direct acolo, valorificand puterea de procesare a motorului SQL. Conceptul de baza ramane acelasi — ceea ce se schimba este unde are loc transformarea.
