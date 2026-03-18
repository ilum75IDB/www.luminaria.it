---
title: "EXPLAIN ANALYZE no basta: como leer realmente un plan de ejecucion PostgreSQL"
description: "Un caso real donde el optimizer eligio un nested loop sobre 2 millones de filas porque las estadisticas estaban desactualizadas. Como leer un plan de ejecucion, encontrar las senales de alarma y actuar."
date: "2025-10-28T10:00:00+01:00"
draft: false
translationKey: "explain_analyze_postgresql"
tags: ["explain", "query-tuning", "execution-plan", "optimizer", "performance"]
categories: ["postgresql"]
image: "explain-analyze-postgresql.cover.jpg"
---

El otro dia un colega me manda una captura de pantalla por Teams. Una query corriendo sobre una tabla de 2 millones de filas, 45 segundos de ejecucion. Me escribe:

> "Hice EXPLAIN ANALYZE, pero no entiendo que esta mal. El plan parece correcto."

Spoiler: el plan no era correcto en absoluto. El optimizer habia elegido un nested loop join donde hacia falta un hash join, y la razon era trivial — estadisticas desactualizadas. Pero para llegar a eso tuve que leer el plan linea por linea, y ahi me di cuenta de que la mayoria de los DBA que conozco usan EXPLAIN ANALYZE como un oraculo binario: si el tiempo es alto, la query es lenta. Fin del analisis.

No. EXPLAIN ANALYZE es una herramienta de diagnostico, no un veredicto. Hay que saber leerlo.

------------------------------------------------------------------------

## 🔧 EXPLAIN, EXPLAIN ANALYZE, EXPLAIN (ANALYZE, BUFFERS): tres cosas distintas

Empecemos por lo basico, porque la confusion esta mas extendida de lo que parece.

**EXPLAIN** solo muestra el plan *estimado*. El optimizer decide que haria, pero no ejecuta nada. Util para entender la estrategia, inutil para entender la realidad.

``` sql
EXPLAIN
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

**EXPLAIN ANALYZE** ejecuta la query y agrega los tiempos reales. Ahora ves cuanto tardo cada nodo, cuantas filas devolvio realmente. Pero falta una pieza.

``` sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

**EXPLAIN (ANALYZE, BUFFERS)** es lo que uso siempre. Agrega la informacion sobre cuantas paginas de disco se leyeron, cuantas estaban en cache (shared hit) y cuantas hubo que cargar desde disco (shared read). Sin BUFFERS estas manejando de noche sin luces.

``` sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

Regla personal: si alguien me manda un EXPLAIN sin BUFFERS, se lo devuelvo.

------------------------------------------------------------------------

## 📖 Anatomia de un nodo: que leer y en que orden

Un plan de ejecucion es un arbol. Cada nodo tiene esta estructura:

``` text
->  Hash Join  (cost=1234.56..5678.90 rows=50000 width=120)
      (actual time=12.345..89.012 rows=48750 loops=1)
      Buffers: shared hit=1200 read=3400
```

Esto es lo que hay que mirar:

**cost** — son dos numeros separados por `..`. El primero es el costo de startup (cuanto antes de devolver la primera fila), el segundo es el costo total estimado. Son unidades arbitrarias del optimizer, no milisegundos. Sirven para comparar planes alternativos, no para medir rendimiento absoluto.

**rows** — las filas estimadas por el optimizer. Comparalas con `actual rows`. Si hay un orden de magnitud de diferencia, encontraste el problema.

**actual time** — tiempo real en milisegundos. Tambien dos valores: startup y total. Atencion al campo `loops`: si loops=10, el tiempo total se multiplica por 10.

**Buffers** — `shared hit` son las paginas encontradas en memoria, `shared read` las leidas desde disco. Si `read` domina, tu working set no cabe en RAM.

------------------------------------------------------------------------

## 🚨 La senal de alarma numero uno: filas estimadas vs filas reales

Vuelvo al caso de mi colega. El plan mostraba:

``` text
->  Nested Loop  (cost=0.87..45678.12 rows=150 width=200)
      (actual time=0.034..44890.123 rows=1950000 loops=1)
```

El optimizer estimaba 150 filas. En realidad llegaron casi 2 millones.

Cuando la estimacion falla por 4 ordenes de magnitud, el plan esta inevitablemente mal. El optimizer eligio un nested loop porque pensaba iterar sobre 150 filas. Un nested loop sobre 150 filas es rapidisimo. Sobre 2 millones es un desastre.

Un hash join o merge join habrian sido la eleccion correcta. Pero el optimizer no podia saberlo con las estadisticas que tenia.

Regla practica: si la relacion entre filas estimadas y reales supera 10x, tienes un problema de estadisticas. Por encima de 100x, el plan esta casi seguramente mal.

------------------------------------------------------------------------

## 🔍 Por que las estadisticas mienten

PostgreSQL mantiene estadisticas sobre las tablas en `pg_statistic` (legibles a traves de `pg_stats`). Estas estadisticas incluyen:

- distribucion de valores (most common values)
- histograma de valores
- numero de valores distintos
- porcentaje de NULL

El optimizer usa esta informacion para estimar la selectividad de cada condicion WHERE y la cardinalidad de cada join.

El problema? Las estadisticas se actualizan con `ANALYZE` — que puede ser manual o gestionado por autovacuum. Pero el autovacuum lanza ANALYZE solo cuando el numero de filas modificadas supera un umbral:

``` text
threshold = autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × n_live_tuples
```

Por defecto: 50 filas + 10% de las filas vivas. En una tabla de 2 millones de filas, se necesitan 200.000 modificaciones antes de que se dispare el ANALYZE automatico.

En el caso de mi colega, la tabla `orders` habia crecido de 500.000 a 2 millones de filas en tres semanas — una importacion masiva desde un sistema legacy. El autovacuum no habia actualizado las estadisticas porque el 10% de 500.000 (el tamano conocido) era 50.000, y las filas habian sido insertadas en lotes que individualmente no superaban el umbral.

Resultado: el optimizer seguia razonando como si la tabla tuviera 500.000 filas con la vieja distribucion de valores.

------------------------------------------------------------------------

## 🛠️ Actualizar estadisticas: lo primero que hacer

La solucion inmediata era obvia:

``` sql
ANALYZE orders;
```

Despues del ANALYZE, volvi a lanzar la query con EXPLAIN (ANALYZE, BUFFERS):

``` text
->  Hash Join  (cost=8500.00..32000.00 rows=1940000 width=200)
      (actual time=120.000..2800.000 rows=1950000 loops=1)
      Buffers: shared hit=28000 read=4500
```

De 45 segundos a menos de 3 segundos. El optimizer habia elegido un hash join, la estimacion de filas era correcta, y el plan era completamente distinto.

Pero no me quede ahi. Si el problema se presento una vez, se va a presentar de nuevo.

------------------------------------------------------------------------

## 📊 default_statistics_target: cuando 100 no alcanza

PostgreSQL recopila 100 valores de muestra por columna por defecto. Para tablas pequenas o con distribucion uniforme, es suficiente. Para tablas grandes con distribucion no uniforme, 100 muestras pueden dar una representacion distorsionada.

En el caso de la tabla `orders`, la columna `customer_id` tenia una distribucion muy sesgada: el 5% de los clientes generaba el 60% de los pedidos. Con 100 muestras, el optimizer no captaba esta asimetria.

La solucion:

``` sql
ALTER TABLE orders
ALTER COLUMN customer_id SET STATISTICS 500;

ANALYZE orders;
```

Despues de subir el target a 500, las estimaciones de cardinalidad del optimizer para los joins con `customers` se volvieron mucho mas precisas.

Regla: si una columna se usa frecuentemente en WHERE o JOIN y tiene distribucion no uniforme, sube el target. 500 es un buen punto de partida. Puedes llegar a 1000, pero mas alla raramente ayuda y ralentiza el ANALYZE mismo.

------------------------------------------------------------------------

## ⚠️ Cuando forzar el planner: enable_nestloop y enable_hashjoin

A veces, incluso con estadisticas actualizadas, el optimizer toma un camino equivocado. Pasa con queries complejas, muchas tablas en join, o cuando la correlacion entre columnas engana las estimaciones.

PostgreSQL ofrece parametros para deshabilitar estrategias especificas:

``` sql
SET enable_nestloop = off;
```

Esto fuerza al optimizer a no usar nested loop. No es una solucion, es un parche diagnostico. Si deshabilitas el nested loop y la query pasa de 45 segundos a 3 segundos, confirmaste que el problema era la eleccion del join. Pero no puedes dejar `enable_nestloop = off` en produccion porque hay mil queries donde el nested loop es la eleccion correcta.

Uso estos parametros solo en dos escenarios:

1. **Diagnostico**: para confirmar cual estrategia de join es el problema
2. **Emergencia**: cuando el negocio esta parado y necesitas que una query critica vuelva a funcionar mientras buscas la solucion real

Despues del diagnostico, el fix correcto siempre es sobre estadisticas, indices o reescritura de la query.

------------------------------------------------------------------------

## 📋 Mi workflow cuando una query es lenta

Despues de treinta anos haciendo este trabajo, mi proceso se ha vuelto casi mecanico:

**1. EXPLAIN (ANALYZE, BUFFERS)** — siempre con BUFFERS. Guardo el output completo, no solo las ultimas lineas.

**2. Busco la discrepancia de filas** — comparo `rows=` estimado con `actual rows=` real en cada nodo. Empiezo por los nodos hoja y subo hacia la raiz. La primera discrepancia significativa es casi siempre la causa.

**3. Reviso las estadisticas** — miro `pg_stats` para las columnas involucradas. Verifico `last_autoanalyze` y `last_analyze` en `pg_stat_user_tables`. Si el ultimo ANALYZE es viejo, lo lanzo y reevaluo.

**4. Evaluo BUFFERS** — si `shared read` es muy alto respecto a `shared hit`, el problema podria ser I/O, no el plan. En ese caso el fix es `shared_buffers` o el working set simplemente no cabe en RAM.

**5. Pruebo alternativas** — si las estadisticas estan actualizadas pero el plan sigue mal, uso `enable_nestloop`, `enable_hashjoin`, `enable_mergejoin` para entender cual estrategia funciona mejor. Luego intento guiar al optimizer hacia esa estrategia con indices o reescritura.

Nada espectacular. Ningun truco magico. Solo lectura sistematica del plan, una linea a la vez.

------------------------------------------------------------------------

## 💬 La leccion de aquel dia

Mi colega, despues de ver la diferencia, me dijo: "Entonces bastaba con un ANALYZE?"

Si y no. En ese caso especifico, si. Pero el punto no es el comando. El punto es saber leer el plan para entender *donde* mirar. EXPLAIN ANALYZE te da los datos. Te toca a ti interpretarlos.

He visto DBA con anos de experiencia lanzar EXPLAIN ANALYZE, mirar el tiempo total al final, y decir "la query es lenta". Es como tomarle la temperatura a un paciente y decir "tiene fiebre". Si, pero de que?

El plan de ejecucion te dice de que. Cada nodo es un organo. Las filas estimadas contra las reales son los valores de laboratorio. Los buffers son las radiografias. Y el ANALYZE es el antibiotico que resuelve el 70% de los casos.

Pero para ese 30% restante, hay que leer. Linea por linea. Nodo por nodo. No hay atajo.
