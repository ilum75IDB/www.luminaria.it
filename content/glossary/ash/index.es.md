---
title: "ASH"
description: "Active Session History — componente Oracle que registra el estado de cada sesion activa una vez por segundo, usado para el diagnostico puntual de problemas de rendimiento."
translationKey: "glossary_ash"
aka: "Active Session History"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**ASH** (Active Session History) es un componente de Oracle Database que muestrea el estado de cada sesion activa una vez por segundo y almacena los datos en un buffer circular en memoria (vista `V$ACTIVE_SESSION_HISTORY`).

## Como funciona

Cada segundo Oracle registra para cada sesion activa:

- SQL en ejecucion (`SQL_ID`)
- Wait event actual
- Programa y modulo llamante
- Plan de ejecucion utilizado (`SQL_PLAN_HASH_VALUE`)

Los datos mas antiguos se descargan automaticamente en las tablas AWR (`DBA_HIST_ACTIVE_SESS_HISTORY`) y se conservan durante el periodo configurado.

## Para que sirve

ASH es el microscopio del DBA: donde AWR muestra promedios sobre intervalos de una hora, ASH permite reconstruir que estaba haciendo una sesion concreta en un instante preciso. Es la herramienta ideal para:

- Identificar quien esta ejecutando un SQL problematico
- Entender cuando empezo un problema (al segundo)
- Correlacionar sesiones, programas y wait events en tiempo real

## Cuando se usa

Se usa cuando el informe AWR ya ha identificado un SQL o un wait event dominante y se necesita detalle: que sesion, que programa, a que hora exacta. La regla empirica: **AWR para entender que cambio, ASH para entender por que**.
