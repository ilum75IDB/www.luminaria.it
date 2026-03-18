---
title: "Wait Event"
description: "Evento de espera registrado por Oracle cada vez que una sesion no puede continuar y debe esperar un recurso — I/O, lock, red o CPU."
translationKey: "glossary_wait_event"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Wait Event** es un indicador diagnostico de Oracle Database que identifica por que una sesion esta esperando en lugar de trabajar activamente. Cada vez que un proceso no puede continuar — porque espera un bloque del disco, un lock, una respuesta de red o un turno de CPU — Oracle registra un wait event especifico.

## Los mas comunes

| Wait Event | Significado |
|---|---|
| `db file sequential read` | Lectura de un solo bloque — tipica de acceso por indice |
| `db file scattered read` | Lectura multi-bloque — tipica de full table scan |
| `log file sync` | Espera del commit en redo log |
| `enq: TX - row lock contention` | Conflicto de lock de fila |
| `direct path read` | Lectura directa (sin pasar por buffer cache) |

## Para que sirven

Los wait events son la base de la metodologia diagnostica de Oracle. Analizando que eventos dominan el DB time (mediante AWR o ASH) se identifica inmediatamente la naturaleza del problema: I/O, contention, CPU o red.

## Donde se encuentran

- **En tiempo real**: `V$SESSION_WAIT`, `V$ACTIVE_SESSION_HISTORY`
- **Historicos**: informes AWR (seccion Top Timed Foreground Events), `DBA_HIST_ACTIVE_SESS_HISTORY`

La regla del DBA: no adivines que ralentiza la base de datos — mira los wait events.
