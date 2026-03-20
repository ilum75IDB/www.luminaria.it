---
title: "Full Table Scan"
description: "Operatie de citire in care Oracle citeste toate blocurile unei tabele de la primul la ultimul, fara a utiliza indecsi."
translationKey: "glossary_full_table_scan"
aka: "TABLE ACCESS FULL"
articles:
  - "/posts/oracle/oracle-awr-ash"
  - "/posts/data-warehouse/partitioning-dwh"
---

**Full Table Scan** (sau TABLE ACCESS FULL) este o operatie in care baza de date citeste toate blocurile de date ale unei tabele, de la inceput la sfarsit, fara a trece prin vreun index.

## Cum functioneaza

Oracle solicita blocuri de pe disc (sau din cache) in secventa, folosind citiri multi-bloc (`db file scattered read`). Fiecare rand din tabela este examinat, indiferent daca indeplineste sau nu criteriile query-ului.

## Cand este o problema

Un full table scan pe o tabela mare este adesea semnul unui index lipsa, al unor statistici depasit sau al unui plan de executie schimbat. In raportul AWR apare ca `db file scattered read` in sectiunea Top Wait Events, cu procente ridicate de DB time.

## Cand este legitim

Pe tabele mici (cateva mii de randuri) sau cand query-ul trebuie efectiv sa citeasca majoritatea datelor, full table scan-ul poate fi mai eficient decat un acces prin index. Problema apare cand Oracle il alege pe tabele cu milioane de randuri pentru a extrage putine inregistrari.

## Cum se identifica

In planul de executie (`EXPLAIN PLAN` sau `DBMS_XPLAN`) apare ca operatie `TABLE ACCESS FULL`. In wait event-urile AWR/ASH se manifesta ca `db file scattered read` dominant.
