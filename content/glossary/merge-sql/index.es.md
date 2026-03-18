---
title: "MERGE"
description: "Instrucción SQL que combina INSERT y UPDATE en una sola operación. En Oracle también conocida como upsert."
translationKey: "glossary_merge_sql"
aka: "UPSERT, MERGE INTO"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

**MERGE** es una instrucción SQL que combina las operaciones de INSERT y UPDATE (y opcionalmente DELETE) en un único statement. Si el registro existe lo actualiza, si no existe lo inserta. Informalmente se conoce como "upsert".

## Sintaxis Oracle

``` sql
MERGE INTO tabla_destino d
USING tabla_fuente s ON (d.clave = s.clave)
WHEN MATCHED THEN UPDATE SET
    d.campo = s.campo
WHEN NOT MATCHED THEN INSERT (clave, campo)
    VALUES (s.clave, s.campo);
```

## Uso en el data warehouse

En el contexto ETL, el MERGE es el mecanismo base para cargar las tablas dimensionales:

- **SCD Tipo 1**: un solo MERGE que actualiza los registros existentes e inserta los nuevos
- **SCD Tipo 2**: el MERGE se usa en la primera fase para cerrar los registros modificados (estableciendo la fecha de fin de validez), seguido de un INSERT para las nuevas versiones

## Disponibilidad

- **Oracle**: soporte completo desde la versión 9i
- **PostgreSQL**: no tiene MERGE nativo hasta la versión 15. La alternativa es `INSERT ... ON CONFLICT` (upsert)
- **MySQL**: usa `INSERT ... ON DUPLICATE KEY UPDATE` como alternativa
- **SQL Server**: soporte completo con sintaxis similar a Oracle
