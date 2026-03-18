---
title: "Snapshot (Oracle)"
description: "Captura puntual de las estadisticas de rendimiento tomada periodicamente por AWR y usada para generar informes diagnosticos comparativos."
translationKey: "glossary_snapshot_oracle"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Snapshot** en Oracle es una captura puntual de las estadisticas de rendimiento de la base de datos almacenada en el repositorio AWR. Por defecto Oracle genera un snapshot cada 60 minutos y los conserva durante 8 dias.

## Como funciona

Cada snapshot registra cientos de metricas: wait events, estadisticas SQL, metricas de memoria (SGA, PGA), I/O por datafile, estadisticas de sistema. La comparacion entre dos snapshots genera el informe AWR, que muestra que cambio entre los dos instantes.

## Snapshots manuales

En situaciones de emergencia se puede generar un snapshot manual para capturar el estado actual:

    EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;

Esto es util cuando se quiere un punto de referencia inmediato — por ejemplo, antes y despues de un deploy — sin esperar al ciclo automatico.

## Gestion

Los snapshots son consultables a traves de la vista `DBA_HIST_SNAPSHOT`. La retencion (cuantos dias conservarlos) y el intervalo (cada cuantos minutos generarlos) se configuran con:

    EXEC DBMS_WORKLOAD_REPOSITORY.modify_snapshot_settings(
      retention => 43200,   -- 30 dias en minutos
      interval  => 30       -- cada 30 minutos
    );

## Por que son importantes

Sin snapshots, no hay AWR. Sin AWR, el diagnostico de un problema de rendimiento se convierte en un ejercicio de intuicion en lugar de un analisis basado en datos. Los snapshots son los cimientos de la observabilidad en Oracle.
