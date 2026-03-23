---
title: "RAC"
description: "Real Application Clusters — tehnologie Oracle care permite mai multor instante sa acceseze simultan aceeasi baza de date, asigurand disponibilitate ridicata si scalabilitate."
translationKey: "glossary_rac"
aka: "Real Application Clusters"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**RAC** (Real Application Clusters) este tehnologia Oracle care permite mai multor instante ale bazei de date sa acceseze simultan acelasi storage partajat. Daca un nod se defecteaza, celelalte continua sa serveasca cererile fara intrerupere — failover-ul este transparent pentru aplicatii.

## Cum functioneaza

Un cluster RAC este compus din doua sau mai multe servere (noduri) conectate printr-o retea privata de mare viteza (interconnect) si storage partajat (de obicei ASM — Automatic Storage Management). Fiecare nod ruleaza propria instanta Oracle, dar toate acceseaza aceleasi datafile-uri.

Protocolul **Cache Fusion** gestioneaza coerenta datelor intre noduri: cand un bloc modificat de un nod este necesar altuia, este transferat direct prin interconnect fara a trece prin disc.

## Disponibilitate ridicata

Daca un nod cade, sesiunile active sunt transferate automat la nodurile ramase prin **TAF** (Transparent Application Failover) sau **Application Continuity**. Timpul de failover depinde de configuratie dar este de obicei de ordinul secundelor.

## Implicatii de licentiere

RAC este o optiune a Enterprise Edition cu costuri semnificative de licentiere. In migrarea cloud, calculul licentelor RAC este unul dintre aspectele cele mai delicate: pe OCI cu BYOL licentele on-premises se reutilizeaza, pe alti furnizori cloud costul se poate multiplica.

## Cand este cu adevarat necesar

RAC este justificat cand sunt necesare disponibilitate ridicata cu failover automat si scalabilitate orizontala. Pentru medii cu putini utilizatori sau cerinte standard de uptime, un singur nod cu Data Guard este adesea o solutie mai simpla si mai putin costisitoare.
