---
title: "MERGE"
description: "Instrucțiune SQL care combină INSERT și UPDATE într-o singură operațiune. În Oracle cunoscută și ca upsert."
translationKey: "glossary_merge_sql"
aka: "UPSERT, MERGE INTO"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**MERGE** este o instrucțiune SQL care combină operațiunile de INSERT și UPDATE (și opțional DELETE) într-un singur statement. Dacă înregistrarea există o actualizează, dacă nu există o inserează. Este numită informal "upsert".

## Sintaxă Oracle

``` sql
MERGE INTO tabel_destinatie d
USING tabel_sursa s ON (d.cheie = s.cheie)
WHEN MATCHED THEN UPDATE SET
    d.camp = s.camp
WHEN NOT MATCHED THEN INSERT (cheie, camp)
    VALUES (s.cheie, s.camp);
```

## Utilizare în data warehouse

În contextul ETL, MERGE este mecanismul de bază pentru încărcarea tabelelor dimensionale:

- **SCD Tip 1**: un singur MERGE care actualizează înregistrările existente și le inserează pe cele noi
- **SCD Tip 2**: MERGE-ul este folosit în prima fază pentru a închide înregistrările modificate (setând data de sfârșit de valabilitate), urmat de un INSERT pentru noile versiuni

## Disponibilitate

- **Oracle**: suport complet din versiunea 9i
- **PostgreSQL**: nu are MERGE nativ până la versiunea 15. Alternativa este `INSERT ... ON CONFLICT` (upsert)
- **MySQL**: folosește `INSERT ... ON DUPLICATE KEY UPDATE` ca alternativă
- **SQL Server**: suport complet cu sintaxă similară cu Oracle
