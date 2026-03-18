---
title: "MERGE"
description: "Istruzione SQL che combina INSERT e UPDATE in un'unica operazione. In Oracle anche nota come upsert."
translationKey: "glossary_merge_sql"
aka: "UPSERT, MERGE INTO"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**MERGE** è un'istruzione SQL che combina le operazioni di INSERT e UPDATE (e opzionalmente DELETE) in un unico statement. Se il record esiste lo aggiorna, se non esiste lo inserisce. È spesso chiamata informalmente "upsert".

## Sintassi Oracle

``` sql
MERGE INTO tabella_destinazione d
USING tabella_sorgente s ON (d.chiave = s.chiave)
WHEN MATCHED THEN UPDATE SET
    d.campo = s.campo
WHEN NOT MATCHED THEN INSERT (chiave, campo)
    VALUES (s.chiave, s.campo);
```

## Uso nel data warehouse

Nel contesto ETL, il MERGE è il meccanismo base per caricare le tabelle dimensionali:

- **SCD Tipo 1**: un singolo MERGE che aggiorna i record esistenti e inserisce quelli nuovi
- **SCD Tipo 2**: il MERGE viene usato nella prima fase per chiudere i record modificati (impostando la data di fine validità), seguito da un INSERT per le nuove versioni

## Disponibilità

- **Oracle**: supporto completo dalla versione 9i
- **PostgreSQL**: non ha MERGE nativo fino alla versione 15. In alternativa si usa `INSERT ... ON CONFLICT` (upsert)
- **MySQL**: usa `INSERT ... ON DUPLICATE KEY UPDATE` come alternativa
- **SQL Server**: supporto completo con sintassi simile a Oracle
