---
title: "Range Partitioning"
description: "Estrategia de particionamiento que divide una tabla en segmentos basados en rangos de valores de una columna, típicamente una fecha."
translationKey: "glossary_range-partitioning"
aka: "Particionamiento por rango"
articles:
  - "/posts/data-warehouse/partitioning-dwh"
  - "/posts/oracle/oracle-partitioning"
---

El **Range Partitioning** (particionamiento por rango) es una estrategia de particionamiento de tablas en la que las filas se distribuyen en particiones diferentes según el valor de una columna respecto a intervalos predefinidos. La columna de particionamiento es casi siempre una fecha en los data warehouses.

## Cómo funciona

Cada partición se define con una cláusula `VALUES LESS THAN` que establece el límite superior del intervalo. Oracle asigna automáticamente cada fila a la partición correcta según el valor de la columna de particionamiento. Si una fila tiene `data_vendita = '2025-03-15'`, se inserta en la partición cuyo rango incluye esa fecha.

## Cuándo se usa

El range partitioning es la elección natural cuando los datos tienen una dimensión temporal dominante — fact tables en data warehouses, tablas de log, tablas de transacciones. La granularidad de la partición (diaria, mensual, trimestral) depende del volumen de inserción y del tipo de queries: particiones demasiado pequeñas generan overhead de gestión, demasiado grandes reducen la eficacia del partition pruning.

## Ventajas operativas

Más allá del rendimiento de las queries, el range partitioning habilita operaciones de gestión del ciclo de vida imposibles en tablas monolíticas: drop instantáneo de una partición (sin DELETE), compresión selectiva de particiones históricas, movimiento a almacenamiento diferente (ILM — Information Lifecycle Management), y exchange partition para cargas masivas sin impacto.
