---
title: "VACUUM y autovacuum: por qué PostgreSQL necesita que alguien limpie"
description: "Una base de datos PostgreSQL de 200 GB con tablas infladas al triple de su tamaño real. El autovacuum estaba activo, pero mal configurado. Cómo diagnosticar el bloat, leer pg_stat_user_tables y hacer tuning sin desactivar nada."
date: "2026-03-24T08:03:00+01:00"
lastmod: "2026-03-24T08:03:00+01:00"
draft: false
translationKey: "vacuum_autovacuum_postgresql"
tags: ["vacuum", "autovacuum", "mvcc", "performance", "bloat"]
categories: ["postgresql"]
image: "vacuum-autovacuum-postgresql.cover.jpg"
---

Hace un par de años me pidieron revisar un PostgreSQL en producción que
"se ralentiza cada semana". Siempre el mismo patrón: el lunes va bien,
el viernes es un desastre. El fin de semana alguien reinicia el
servicio y se empieza de nuevo.

Base de datos de unos 200 GB. Las tablas principales ocupaban casi el
triple del espacio efectivo de los datos. Queries cayendo en sequential
scan donde no deberían. Tiempos de respuesta que subían día tras día.

El autovacuum estaba activo. Nadie lo había desactivado. Pero nadie lo
había configurado tampoco.

------------------------------------------------------------------------

## 🧠 MVCC: por qué PostgreSQL genera "basura"

Para entender el problema hay que dar un paso atrás. PostgreSQL usa
MVCC — Multi-Version Concurrency Control. Cada vez que haces un UPDATE,
la base de datos no sobreescribe la fila original. Crea una nueva
versión y marca la vieja como "muerta".

Lo mismo para los DELETE: la fila no se elimina físicamente. Se marca
como no visible para las nuevas transacciones.

Estas filas muertas se llaman **dead tuples**. Y se quedan ahí, dentro
de las páginas de datos, ocupando espacio en disco y ralentizando los
escaneos.

Es el precio que PostgreSQL paga por tener aislamiento transaccional sin
bloqueos exclusivos en las lecturas. Un precio razonable — siempre que
alguien pase a limpiar.

------------------------------------------------------------------------

## 🔧 VACUUM: qué hace realmente

El comando `VACUUM` hace una cosa simple: recupera el espacio ocupado
por los dead tuples y lo hace reutilizable para nuevas inserciones.

No devuelve espacio al sistema operativo. No reorganiza la tabla. No
compacta nada. Marca las páginas como reescribibles.

``` sql
VACUUM reporting.transactions;
```

Esto basta en la mayoría de los casos. VACUUM es ligero, no bloquea las
escrituras y puede funcionar en paralelo con las queries normales.

### ¿Y `VACUUM FULL`?

`VACUUM FULL` es otra bestia. Reescribe físicamente la tabla entera,
eliminando todo el espacio muerto. Devuelve espacio al sistema de
archivos.

Pero el coste es brutal: **toma un bloqueo exclusivo** sobre la tabla
durante toda la operación. Nadie lee, nadie escribe. En tablas grandes
hablamos de minutos u horas.

``` sql
VACUUM FULL reporting.transactions;
```

En producción, `VACUUM FULL` debe usarse muy raramente. En emergencias.
Y siempre fuera de horario.

------------------------------------------------------------------------

## ⚙️ Autovacuum: el conserje silencioso

PostgreSQL tiene un daemon que ejecuta el VACUUM automáticamente:
el autovacuum.

Se activa cuando una tabla acumula suficientes dead tuples. El umbral
se calcula así:

```
vacuum threshold = autovacuum_vacuum_threshold
                 + autovacuum_vacuum_scale_factor × n_live_tup
```

Los valores por defecto:

- `autovacuum_vacuum_threshold`: **50** dead tuples
- `autovacuum_vacuum_scale_factor`: **0.2** (20%)

Traducido: en una tabla con 10 millones de filas, el autovacuum arranca
cuando los dead tuples superan **2.000.050**. Dos millones de filas
muertas antes de que alguien haga limpieza.

Para una tabla con 500.000 updates al día, eso significa que el
autovacuum se activa quizá cada 4 días. Mientras tanto el bloat crece,
los escaneos se ralentizan, los índices se hinchan.

Por eso el lunes todo iba bien y el viernes era un desastre.

------------------------------------------------------------------------

## 📊 Diagnóstico: leer pg_stat_user_tables

Lo primero que hay que hacer cuando sospechas un problema de vacuum es
consultar `pg_stat_user_tables`:

``` sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    autovacuum_count,
    vacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

En el caso de mi cliente, la situación era esta:

``` text
relname            | n_live_tup | n_dead_tup | dead_pct | last_autovacuum
-------------------+------------+------------+----------+------------------
transactions       | 12.400.000 |  3.800.000 |   23,5%  | hace 3 días
order_lines        |  8.200.000 |  2.100.000 |   20,4%  | hace 4 días
inventory_moves    |  5.600.000 |  1.900.000 |   25,3%  | hace 5 días
```

Casi una cuarta parte de las filas estaban muertas. El autovacuum
funcionaba, pero con demasiada poca frecuencia para mantener el ritmo.

------------------------------------------------------------------------

## 🎯 Tuning: adaptar el autovacuum a la realidad

El truco no es desactivar el autovacuum. Nunca. El truco es configurarlo
para las tablas que lo necesitan.

PostgreSQL permite establecer parámetros de autovacuum **por tabla
individual**:

``` sql
ALTER TABLE reporting.transactions SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000
);
```

Con esta configuración, el autovacuum arranca después de 1.000 + 1% de
las filas vivas de dead tuples. En 12 millones de filas, se activa con
~121.000 dead tuples en vez de 2 millones.

### cost_delay: no estrangules al vacuum

Otro parámetro crítico es `autovacuum_vacuum_cost_delay`. Controla
cuánto el vacuum "se frena a sí mismo" para no sobrecargar el I/O.

El valor por defecto es 2 milisegundos. En servidores modernos con SSD,
es demasiado conservador. Reducirlo a 0 o 1 ms permite que el vacuum
termine antes:

``` sql
ALTER TABLE reporting.transactions SET (
    autovacuum_vacuum_cost_delay = 0
);
```

### max_workers

El valor por defecto es 3 workers de autovacuum. Si tienes decenas de
tablas con mucho tráfico, 3 workers no bastan. Evalúa subirlos a 5–6,
monitorizando el impacto en CPU e I/O:

``` text
-- en postgresql.conf
autovacuum_max_workers = 5
```

------------------------------------------------------------------------

## 📏 Medir el bloat

¿Cómo sabes cuánto espacio están desperdiciando tus tablas?

La consulta clásica usa `pgstattuple`:

``` sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    pg_size_pretty(pg_total_relation_size('reporting.transactions')) AS total_size,
    pg_size_pretty(pg_total_relation_size('reporting.transactions')
                   - pg_relation_size('reporting.transactions')) AS index_size,
    *
FROM pgstattuple('reporting.transactions');
```

Los campos clave: `dead_tuple_percent` y `free_space`. Si dead_tuple
supera el 20–30%, la tabla tiene un problema serio.

Una alternativa menos precisa pero más ligera es estimar el bloat ratio
comparando `pg_class.relpages` con las filas estimadas — hay consultas
consolidadas en la comunidad para esto (la clásica "bloat estimation
query" de PostgreSQL Experts).

------------------------------------------------------------------------

## 🛠️ Cuando VACUUM no basta: pg_repack

Si el bloat ya se ha ido de las manos — tablas al 50–70% de espacio
muerto — el VACUUM normal no recupera todo. Libera los dead tuples,
pero el espacio fragmentado permanece.

`VACUUM FULL` funciona, pero bloquea todo.

La alternativa en producción es **pg_repack**: reconstruye la tabla
online, sin bloqueos exclusivos prolongados.

``` bash
pg_repack -d mydb -t reporting.transactions
```

No es una solución para usar cada semana. Es la cura de choque para
cuando la situación ya ha degenerado. La verdadera solución es no
llegar a ese punto, con un autovacuum bien configurado.

------------------------------------------------------------------------

## 💬 El principio

Desactivar el autovacuum es lo peor que puedes hacerle a un PostgreSQL
en producción. Lo he visto hacer "porque ralentiza las queries durante
el día". Claro, porque mientras tanto el bloat te está devorando la
base de datos desde dentro.

El autovacuum con los valores por defecto de PostgreSQL está pensado
para una base de datos genérica. Ninguna base de datos en producción es
genérica. Cada tabla tiene su patrón de escritura, su volumen, su ritmo.

Tres cosas para llevarse:

1. Revisa `pg_stat_user_tables` con regularidad. Si `n_dead_tup` crece
   más rápido de lo que el autovacuum puede limpiar, tienes un
   problema.

2. Configura `scale_factor` y `threshold` para las tablas de alto
   tráfico. No existe una configuración universal.

3. No esperes a que el bloat llegue al 50% para actuar. A ese punto
   las opciones son pocas y todas dolorosas.

Las bases de datos no se mantienen solas. Ni siquiera las que tienen un
daemon que lo intenta.

------------------------------------------------------------------------

## Glosario

**[VACUUM](/es/glossary/vacuum/)** — Comando PostgreSQL que recupera el espacio ocupado por dead tuples, haciéndolo reutilizable para nuevas inserciones sin devolverlo al sistema operativo.

**[MVCC](/es/glossary/mvcc/)** — Multi-Version Concurrency Control — modelo de concurrencia de PostgreSQL que mantiene múltiples versiones de las filas para garantizar aislamiento transaccional sin locks exclusivos en las lecturas.

**[Dead Tuple](/es/glossary/dead-tuple/)** — Fila obsoleta en una tabla PostgreSQL, marcada como ya no visible después de un UPDATE o DELETE pero aún no eliminada físicamente del disco.

**[Autovacuum](/es/glossary/autovacuum/)** — Daemon de PostgreSQL que ejecuta automáticamente VACUUM y ANALYZE en las tablas cuando el número de dead tuples supera un umbral configurable.

**[Bloat](/es/glossary/bloat/)** — Espacio muerto acumulado en una tabla o índice PostgreSQL debido a dead tuples no eliminados, que hincha el tamaño en disco y degrada el rendimiento.
