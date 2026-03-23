---
title: "Transport Lag"
description: "Retardo en la transmision de los redo logs desde la base de datos primary al standby en una configuracion Data Guard. Indicador critico de la salud de la replicacion."
translationKey: "glossary_transport_lag"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

El **transport lag** es el retardo entre el momento en que la base de datos primary genera un redo log y el momento en que ese redo log es recibido por la base de datos standby en una configuracion Oracle Data Guard. Es uno de los indicadores mas importantes para evaluar la salud de la replicacion.

## Como se mide

El transport lag se monitorea con una consulta sobre la vista `V$DATAGUARD_STATS` o mediante Data Guard Broker:

    DGMGRL> SHOW DATABASE 'standby_db' 'TransportLagTarget';

El valor se expresa en formato tiempo (ej. `+00 00:00:03` = 3 segundos de retardo). Un transport lag de pocos segundos es normal en modo Maximum Performance; un lag que crece constantemente indica un problema de ancho de banda o de redo generate rate.

## Diferencia con Apply Lag

| Metrica | Que mide |
|---------|----------|
| **Transport Lag** | Retardo en la transmision de los redo del primary al standby |
| **Apply Lag** | Retardo en la aplicacion de los redo en el standby tras la recepcion |

El transport lag depende de la red (ancho de banda, latencia); el apply lag depende de los recursos del standby (CPU, I/O). En las migraciones cross-site, el transport lag es el cuello de botella mas comun.

## Impacto en las migraciones

Durante una migracion con Data Guard cross-site, el transport lag debe monitorearse cuidadosamente durante las fases de carga maxima (batch nocturnos, picos de actividad). Un redo generate rate que supera la capacidad de la VPN produce un transport lag creciente. Antes del cutover, el transport lag debe estar cerca de cero para garantizar que el switchover ocurra sin perdida de datos.
