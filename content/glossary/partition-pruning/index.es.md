---
title: "Partition Pruning"
description: "Mecanismo automático de Oracle que excluye las particiones no relevantes durante la ejecución de una query, leyendo solo las particiones que contienen datos correspondientes al predicado."
translationKey: "glossary_partition-pruning"
articles:
  - "/posts/oracle/oracle-partitioning"
  - "/posts/data-warehouse/partitioning-dwh"
---

El **Partition Pruning** es el mecanismo por el cual Oracle, durante la ejecución de una query sobre una tabla particionada, identifica y excluye automáticamente las particiones que no pueden contener datos relevantes para el predicado de la query.

## Cómo funciona

Cuando una query incluye un predicado sobre la columna de partición (ej. `WHERE data_movimento BETWEEN ...`), Oracle consulta los metadatos de las particiones y determina cuáles contienen datos en el rango solicitado. Solo esas particiones se leen. En el plan de ejecución aparece como `PARTITION RANGE SINGLE` o `PARTITION RANGE ITERATOR`.

## Para qué sirve

En una tabla de 380 GB con particiones mensuales, una query sobre un solo mes lee solo ~4 GB en lugar de la tabla entera. El pruning transforma un full table scan de pesadilla en un full partition scan manejable, reduciendo el I/O en un 99%.

## Cuándo se usa

El pruning es automático, pero solo funciona con predicados directos sobre la columna de partición. Aplicar funciones a la columna (`TRUNC(fecha)`, `TO_CHAR(fecha)`) desactiva el pruning y fuerza a Oracle a leer todas las particiones. Verificar siempre con `EXPLAIN PLAN` que el pruning esté activo.
