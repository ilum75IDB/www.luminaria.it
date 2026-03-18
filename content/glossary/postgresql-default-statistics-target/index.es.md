---
title: "default_statistics_target"
description: "El parametro PostgreSQL que controla cuantas muestras recopila ANALYZE para estimar la distribucion de datos en cada columna."
translationKey: "glossary_postgresql_default_statistics_target"
aka: "default_statistics_target (PostgreSQL)"
articles:
  - "/posts/postgresql/explain-analyze-postgresql"
---

**default_statistics_target** es el parametro PostgreSQL que define el numero de muestras recopiladas por el comando `ANALYZE` para construir las estadisticas de cada columna. El valor por defecto es 100.

## Como funciona

PostgreSQL muestrea un cierto numero de valores por cada columna y los usa para construir dos estructuras:

- **Most common values (MCV)**: la lista de los valores mas frecuentes, con sus respectivas frecuencias
- **Histograma**: la distribucion de los valores restantes, dividida en buckets de igual poblacion

El parametro `default_statistics_target` determina cuantos elementos tendran estas estructuras. Con el valor 100 (por defecto), el histograma tendra 100 buckets y la lista MCV contendra hasta 100 valores.

## Cuando aumentarlo

Para tablas pequenas o con distribucion uniforme, 100 muestras son suficientes. Para tablas grandes con distribucion asimetrica (skewed) — donde pocos valores dominan la mayoria de las filas — 100 muestras pueden dar una representacion distorsionada, llevando al optimizer a estimaciones de cardinalidad erroneas.

Se puede aumentar el target a nivel de columna individual:

    ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
    ANALYZE orders;

Valores entre 500 y 1000 mejoran sensiblemente la calidad de las estimaciones en columnas con distribucion no uniforme.

## Limites practicos

Por encima de 1000 el beneficio es marginal y el `ANALYZE` mismo se vuelve mas lento, porque necesita muestrear mas filas y construir estructuras mas grandes. Es un ajuste fino: hay que aplicarlo solo a las columnas que efectivamente causan estimaciones erroneas, no a todas las columnas de todas las tablas.
