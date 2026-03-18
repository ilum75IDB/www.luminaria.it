---
title: "Data Warehouse"
description: "Sistem centralizat de colectare și istoricizare a datelor din surse diverse, proiectat pentru analiză și suport al deciziilor de afaceri."
translationKey: "glossary_data-warehouse"
articles:
  - "/posts/project-management/4-milioni-nessun-software"
---

Un **Data Warehouse** (DWH) este un sistem de stocare a datelor proiectat specific pentru analiză, raportare și suport al deciziilor de afaceri. Spre deosebire de bazele de date operaționale (OLTP), un DWH colectează date din surse multiple, le transformă și le organizează în structuri optimizate pentru interogări analitice.

## Cum funcționează

Datele sunt extrase din sistemele sursă (aplicații de gestiune, CRM, ERP), transformate prin procese ETL care le curăță, normalizează și îmbogățesc, și în final încărcate în DWH. Modelul de date tipic este star schema: o tabelă de fapte centrală cu măsurile numerice conectată la tabele dimensionale care descriu contextul (timp, client, produs, geografie).

## La ce folosește

Un DWH permite răspunsul la întrebări de business pe care sistemele operaționale nu le pot gestiona: tendințe istorice, analize comparative între perioade, agregări cross-system, KPI-uri de afaceri. Separă sarcina analitică de cea tranzacțională, prevenind impactul interogărilor de raportare asupra performanței aplicațiilor operative.

## Când se folosește

Un DWH este necesar când o companie trebuie să integreze date din surse diverse pentru a produce analize consolidate. Complexitatea și costurile depind de numărul sistemelor sursă, volumul datelor și frecvența de actualizare necesară.
