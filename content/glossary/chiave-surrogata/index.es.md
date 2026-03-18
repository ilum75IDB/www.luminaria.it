---
title: "Clave subrogada"
description: "Identificador numérico generado por el data warehouse, distinto de la clave natural del sistema fuente. Imprescindible en la SCD Tipo 2."
translationKey: "glossary_chiave_surrogata"
aka: "Surrogate key"
articles:
  - "/posts/data-warehouse/scd-tipo-2"
---

La **clave subrogada** (surrogate key) es un identificador numérico secuencial generado internamente por el data warehouse, sin ningún significado de negocio. Es distinta de la clave natural — la que proviene del sistema fuente (ej. el código de cliente, el número de empleado).

## Por qué es necesaria

En la SCD Tipo 2, el mismo cliente puede tener múltiples filas en la tabla dimensional — una por cada versión histórica. La clave natural (`cliente_id`) ya no es única, así que se necesita un identificador que distinga cada versión individual: la clave subrogada (`cliente_key`).

## Cómo funciona

Se genera típicamente con una sequence (Oracle) o una columna SERIAL/IDENTITY (PostgreSQL, MySQL). Nunca se expone a los usuarios finales y no tiene significado fuera del data warehouse.

La tabla de hechos usa la clave subrogada como clave foránea, apuntando a la versión específica de la dimensión que era vigente en el momento del hecho. Esto garantiza que cada transacción esté asociada al contexto dimensional correcto para ese momento en el tiempo.

## Ventajas

- Permite el versionado de las dimensiones (SCD Tipo 2)
- Los joins entre hechos y dimensiones son sobre enteros, por lo tanto rápidos
- Aísla el DWH de los cambios en las claves de los sistemas fuente
- Soporta la carga desde múltiples fuentes con claves naturales potencialmente duplicadas
