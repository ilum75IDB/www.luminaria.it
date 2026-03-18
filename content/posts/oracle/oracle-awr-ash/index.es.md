---
title: "AWR, ASH y los 10 minutos que salvaron un go-live"
description: "Viernes por la noche, víspera de un go-live. El rendimiento se desploma. Con AWR y ASH encontré un full table scan oculto en un procedimiento almacenado en menos de diez minutos — y el paso a producción siguió adelante."
date: "2026-02-10T10:00:00+01:00"
draft: false
translationKey: "oracle_awr_ash"
tags: ["awr", "ash", "performance", "tuning", "go-live", "diagnostic"]
categories: ["oracle"]
image: "oracle-awr-ash.cover.jpg"
---

Viernes, 18:40. Ya tenía la chaqueta puesta, listo para salir. El teléfono vibra. Es el project manager.

"Ivan, tenemos un problema. El sistema va lentísimo. Mañana por la mañana es el go-live."

No es la primera vez que recibo una llamada así. Pero el tono era diferente. No era la queja genérica sobre la lentitud. Era pánico.

Me reconecto por VPN, abro una sesión en la base de datos Oracle 19c del cliente. Lo primero que hago es una comprobación rápida:

``` sql
SELECT metric_name, value
FROM   v$sysmetric
WHERE  metric_name IN ('Database CPU Time Ratio',
                       'Database Wait Time Ratio',
                       'Average Active Sessions');
```

**CPU Time Ratio**: 12%. En condiciones normales estaba por encima del 80%.

**Average Active Sessions**: 47. En un servidor con 16 cores.

Cuarenta y siete sesiones activas. La base de datos se estaba ahogando.

------------------------------------------------------------------------

## 🔥 Los síntomas

El equipo de desarrollo había completado el último deploy del código aplicativo esa tarde. Todo parecía funcionar en las pruebas. Pero cuando lanzaron el batch de verificación pre-go-live — el que simula la carga de producción — los tiempos de respuesta explotaron.

Las queries que normalmente tardaban 2-3 segundos necesitaban 45. Los batches que terminaban en 20 minutos seguían en ejecución después de una hora. Los {{< glossary term="wait-event" >}}wait events{{< /glossary >}} dominantes eran `db file sequential read` y `db file scattered read` — señal inequívoca de I/O físico masivo.

Algo estaba leyendo cantidades enormes de datos del disco. Algo que antes no estaba.

------------------------------------------------------------------------

## 📊 AWR: la fotografía del problema

{{< glossary term="awr" >}}AWR{{< /glossary >}} — Automatic Workload Repository — es la herramienta de diagnóstico más potente que Oracle pone a disposición. Cada hora, Oracle toma una instantánea ({{< glossary term="snapshot-oracle" >}}snapshot{{< /glossary >}}) de las estadísticas de rendimiento y la conserva en el repositorio interno. Comparando dos snapshots, obtienes un informe que te dice exactamente qué ocurrió en ese período.

Generé un snapshot manual para capturar la situación actual:

``` sql
EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;
```

Luego busqué los snapshots disponibles:

``` sql
SELECT snap_id, begin_interval_time, end_interval_time
FROM   dba_hist_snapshot
WHERE  begin_interval_time > SYSDATE - 1/6
ORDER BY snap_id DESC;
```

Tenía un snapshot de las 18:00 (antes del problema visible) y el que acababa de crear a las 18:45. Generé el informe AWR:

``` sql
SELECT output
FROM   TABLE(DBMS_WORKLOAD_REPOSITORY.awr_report_text(
         l_dbid     => (SELECT dbid FROM v$database),
         l_inst_num => 1,
         l_bid      => 4523,
         l_eid      => 4524
       ));
```

### Qué decía el informe

La sección **Top 5 Timed Foreground Events** era elocuente:

| Event | Waits | Time (s) | % DB time |
|---|---|---|---|
| db file scattered read | 1.247.832 | 3.847 | 58,2% |
| db file sequential read | 423.109 | 1.205 | 18,2% |
| CPU + Wait for CPU | — | 892 | 13,5% |
| log file sync | 12.445 | 287 | 4,3% |
| direct path read | 8.221 | 198 | 3,0% |

`db file scattered read` al 58%. Son {{< glossary term="full-table-scan" >}}full table scans{{< /glossary >}}. Algo estaba leyendo tablas enteras, bloque a bloque, sin usar índices.

La sección **SQL ordered by Elapsed Time** mostraba un solo SQL_ID que consumía el 71% del tiempo total de la base de datos: `g4f2h8k1nw3z9`.

Ahora sabía qué buscar.

------------------------------------------------------------------------

## 🔍 ASH: el microscopio

AWR me había dado la fotografía general. Pero necesitaba entender **cuándo** empezó ese SQL, **quién** lo ejecutaba, y **qué programa** lo había lanzado.

{{< glossary term="ash" >}}ASH{{< /glossary >}} — Active Session History — registra el estado de cada sesión activa una vez por segundo. Es el microscopio del DBA: donde AWR te muestra promedios de una hora, ASH te muestra qué ocurría segundo a segundo.

``` sql
SELECT sample_time,
       session_id,
       sql_id,
       sql_plan_hash_value,
       event,
       program,
       module
FROM   v$active_session_history
WHERE  sql_id = 'g4f2h8k1nw3z9'
  AND  sample_time > SYSDATE - 1/24
ORDER BY sample_time DESC;
```

Los resultados eran claros:

- **Program**: `JDBC Thin Client` — la aplicación Java del batch
- **Module**: `BatchVerificaProduzione`
- **Event**: `db file scattered read` en el 92% de las muestras
- **Primera ocurrencia**: 17:12 — exactamente después del deploy de la tarde
- **SQL_PLAN_HASH_VALUE**: `2891047563`

El plan de ejecución había cambiado. Antes del deploy, esa query usaba un plan diferente.

------------------------------------------------------------------------

## 🧩 El plan de ejecución

Recuperé el plan actual:

``` sql
SELECT *
FROM   TABLE(DBMS_XPLAN.display_awr(
         sql_id          => 'g4f2h8k1nw3z9',
         plan_hash_value => 2891047563
       ));
```

El resultado me hizo entender el problema de inmediato:

```
---------------------------------------------------------------------------
| Id | Operation            | Name            | Rows  | Cost  |
---------------------------------------------------------------------------
|  0 | SELECT STATEMENT     |                 |       | 48721 |
|  1 |  HASH JOIN           |                 | 2.1M  | 48721 |
|  2 |   TABLE ACCESS FULL  | MOVIMENTI_TEMP  | 2.1M  | 41893 |
|  3 |   INDEX RANGE SCAN   | IDX_CLIENTI_PK  |     1 |     2 |
---------------------------------------------------------------------------
```

**TABLE ACCESS FULL sobre MOVIMENTI_TEMP**. Una tabla temporal con 2,1 millones de filas, leída por completo cada vez. Sin índice. Sin filtro eficaz.

Verifiqué qué existía antes del deploy consultando el plan anterior en AWR:

``` sql
SELECT plan_hash_value, timestamp
FROM   dba_hist_sql_plan
WHERE  sql_id = 'g4f2h8k1nw3z9'
ORDER BY timestamp;
```

El plan anterior (hash `1384726091`) usaba un `INDEX RANGE SCAN` sobre un índice que — descubrimiento — **había sido eliminado durante el deploy**. El script de migración incluía un `DROP TABLE MOVIMENTI_TEMP` seguido de una recreación, pero sin recrear el índice.

------------------------------------------------------------------------

## ⚡ La solución

Diez minutos. Desde el momento en que me conecté hasta que identifiqué la causa. No por habilidad — por las herramientas.

La corrección era sencilla:

``` sql
CREATE INDEX idx_movimenti_temp_cliente
ON movimenti_temp (id_cliente, data_movimento)
TABLESPACE idx_data;
```

Después de crear el índice, forcé un re-parse de la query:

``` sql
EXEC DBMS_SHARED_POOL.purge('g4f2h8k1nw3z9', 'C');
```

Pedí al equipo que relanzara el batch. Tiempo de ejecución: 18 minutos. Idéntico a las pruebas anteriores.

El go-live del sábado por la mañana se realizó con normalidad.

------------------------------------------------------------------------

## 📋 AWR vs ASH: cuándo usar cada uno

Después de aquel episodio formalicé una regla que siempre sigo:

| Característica | AWR | ASH |
|---|---|---|
| Granularidad | Snapshots por hora (configurables) | Muestra cada segundo |
| Profundidad histórica | Hasta 30 días (por defecto 8) | 1 hora en memoria, luego en AWR |
| Caso de uso principal | Análisis de tendencias, comparación de períodos | Diagnóstico puntual, aislamiento de SQL |
| Vista principal | `DBA_HIST_*` | `V$ACTIVE_SESSION_HISTORY` |
| Vista histórica | — | `DBA_HIST_ACTIVE_SESS_HISTORY` |
| Licencia requerida | Diagnostic Pack | Diagnostic Pack |
| Salida típica | Informe HTML/texto | Queries ad hoc |

La regla empírica: **AWR para entender qué cambió, ASH para entender por qué**.

AWR te dice: "Entre las 17:00 y las 18:00, el 58% del tiempo de la base de datos se gastó en full table scans." ASH te dice: "A las 17:12:34, la sesión 847 estaba ejecutando la query g4f2h8k1nw3z9 con un full table scan sobre MOVIMENTI_TEMP, lanzada por el programa BatchVerificaProduzione."

Son complementarios. Usar solo uno es como diagnosticar un problema mirando solo el TAC o solo los análisis de sangre.

------------------------------------------------------------------------

## 🛡️ Las queries que todo DBA debería tener preparadas

A lo largo de los años he construido un conjunto de queries diagnósticas que siempre tengo a mano. Las comparto porque en una emergencia no hay tiempo para escribirlas desde cero.

### Top SQL por tiempo de ejecución (última hora)

``` sql
SELECT sql_id,
       COUNT(*) AS samples,
       ROUND(COUNT(*) / 60, 1) AS est_minutes,
       MAX(event) AS top_event,
       MAX(program) AS program
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/24
  AND  sql_id IS NOT NULL
GROUP BY sql_id
ORDER BY samples DESC
FETCH FIRST 10 ROWS ONLY;
```

### Distribución de wait events para un SQL específico

``` sql
SELECT event,
       COUNT(*) AS samples,
       ROUND(COUNT(*) * 100 / SUM(COUNT(*)) OVER(), 1) AS pct
FROM   v$active_session_history
WHERE  sql_id = '&sql_id'
  AND  sample_time > SYSDATE - 1/24
GROUP BY event
ORDER BY samples DESC;
```

### Comparación de planes de ejecución en el tiempo

``` sql
SELECT plan_hash_value,
       MIN(timestamp) AS first_seen,
       MAX(timestamp) AS last_seen,
       COUNT(*) AS executions_in_awr
FROM   dba_hist_sqlstat
WHERE  sql_id = '&sql_id'
GROUP BY plan_hash_value
ORDER BY first_seen;
```

------------------------------------------------------------------------

## 🎯 Qué aprendí aquella noche

Tres lecciones que llevo conmigo.

**Primera**: el deploy no es solo código. Es también estructura. Cuando liberas en producción, debes verificar que índices, constraints, estadísticas y grants sean coherentes con lo que había antes. Un script que hace `DROP TABLE` y `CREATE TABLE` sin recrear los índices es una bomba de relojería.

**Segunda**: AWR y ASH no son herramientas para DBAs senior. Son herramientas de primera línea, como un desfibrilador. Debes saber usarlas antes de necesitarlas, no durante la emergencia.

**Tercera**: diez minutos de diagnóstico correcto valen más que tres horas de intentos a ciegas. Cuando el sistema está de rodillas, la tentación es reiniciar, matar sesiones, meter más recursos. Pero sin saber qué está pasando, estás disparando en la oscuridad.

Esa noche salí de la oficina a las 19:20. Cuarenta minutos después de la llamada. Al día siguiente el go-live arrancó sin contratiempos, y el lunes el sistema funcionaba con normalidad.

No soy un héroe. Solo usé las herramientas correctas.

------------------------------------------------------------------------

## Glosario

**[AWR](/es/glossary/awr/)** — Automatic Workload Repository. Componente integrado en Oracle que recopila estadisticas de rendimiento mediante snapshots periodicos y genera informes diagnosticos comparativos.

**[ASH](/es/glossary/ash/)** — Active Session History. Componente Oracle que muestrea el estado de cada sesion activa una vez por segundo, almacenandolo en memoria y luego en AWR. Es el microscopio del DBA para el diagnostico puntual.

**[Full Table Scan](/es/glossary/full-table-scan/)** — Operacion de lectura en la que Oracle lee todos los bloques de una tabla sin usar indices. En los wait events aparece como `db file scattered read`.

**[Wait Event](/es/glossary/wait-event/)** — Evento de espera registrado por Oracle cada vez que una sesion no puede continuar porque espera un recurso (I/O, lock, CPU, red). El analisis de wait events es la base de la metodologia diagnostica de Oracle.

**[Snapshot](/es/glossary/snapshot-oracle/)** — Captura puntual de las estadisticas de rendimiento tomada periodicamente por AWR (por defecto cada 60 minutos). La comparacion entre dos snapshots genera el informe AWR.
