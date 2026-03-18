---
title: "Full Table Scan"
description: "Operacion de lectura en la que Oracle lee todos los bloques de una tabla del primero al ultimo, sin utilizar indices."
translationKey: "glossary_full_table_scan"
aka: "TABLE ACCESS FULL"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Full Table Scan** (o TABLE ACCESS FULL) es una operacion en la que la base de datos lee todos los bloques de datos de una tabla, de principio a fin, sin pasar por ningun indice.

## Como funciona

Oracle solicita bloques del disco (o de la cache) en secuencia, usando lecturas multi-bloque (`db file scattered read`). Cada fila de la tabla es examinada, independientemente de si cumple o no los criterios de la query.

## Cuando es un problema

Un full table scan sobre una tabla grande es frecuentemente senal de un indice faltante, estadisticas obsoletas o un plan de ejecucion modificado. En el informe AWR aparece como `db file scattered read` en la seccion Top Wait Events, con porcentajes elevados de DB time.

## Cuando es legitimo

En tablas pequenas (unos pocos miles de filas) o cuando la query realmente necesita leer la mayor parte de los datos, el full table scan puede ser mas eficiente que un acceso por indice. El problema surge cuando Oracle lo elige en tablas con millones de filas para extraer pocos registros.

## Como se identifica

En el plan de ejecucion (`EXPLAIN PLAN` o `DBMS_XPLAN`) aparece como operacion `TABLE ACCESS FULL`. En los wait events de AWR/ASH se manifiesta como `db file scattered read` dominante.
