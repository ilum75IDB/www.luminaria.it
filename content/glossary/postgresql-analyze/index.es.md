---
title: "ANALYZE"
description: "El comando PostgreSQL que actualiza las estadisticas de las tablas utilizadas por el optimizer para elegir el plan de ejecucion."
translationKey: "glossary_postgresql_analyze"
aka: "ANALYZE (PostgreSQL)"
articles:
  - "/posts/postgresql/explain-analyze-postgresql"
---

**ANALYZE** es el comando PostgreSQL que recopila estadisticas sobre la distribucion de los datos en las tablas y las almacena en el catalogo `pg_statistic` (legible a traves de la vista `pg_stats`). El optimizer usa estas estadisticas para estimar la cardinalidad — cuantas filas devolvera cada operacion — y elegir el plan de ejecucion mas eficiente.

## Que recopila

Las estadisticas recopiladas por ANALYZE incluyen:

- **Most common values**: los valores mas frecuentes por cada columna y su porcentaje
- **Histogramas de distribucion**: como se distribuyen los valores restantes
- **Numero de valores distintos**: cuantos valores unicos tiene cada columna
- **Porcentaje de NULL**: cuantas filas tienen valor NULL por cada columna

La calidad de estas estadisticas depende del numero de muestras recopiladas, controlado por el parametro `default_statistics_target`.

## Por que es critico

Sin estadisticas actualizadas, el optimizer se ve obligado a adivinar. Las estimaciones erroneas llevan a planes de ejecucion desastrosos — como elegir un nested loop sobre millones de filas pensando que son cientos, o ignorar un indice perfectamente adecuado.

## Cuando ejecutarlo

PostgreSQL ejecuta ANALYZE automaticamente a traves del autovacuum, pero el umbral por defecto (50 filas + 10% de las filas vivas) puede ser demasiado alto para tablas que crecen rapidamente. Situaciones donde un ANALYZE manual es necesario:

- Despues de importaciones masivas o bulk load
- Despues de cambios significativos en la distribucion de los datos
- Cuando `EXPLAIN ANALYZE` muestra estimaciones de cardinalidad muy diferentes de las filas reales
- Despues de modificar el `default_statistics_target` de una columna
