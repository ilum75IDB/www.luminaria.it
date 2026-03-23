---
title: "When a LIKE '%value%' Slows Everything Down: A Real PostgreSQL Optimization Case"
description: "A real-world PostgreSQL performance case where a LIKE '%value%' caused a full scan and degraded response times. Analysis, execution plan, and a scalable indexing strategy."
date: "2026-01-06T10:00:00+01:00"
lastmod: "2026-02-26T09:34:00+01:00"
draft: false
translationKey: "like_optimization_postgresql"
tags: ["query-tuning", "performance", "indexes", "pg_trgm"]
categories: ["postgresql"]
image: "like-optimization-postgresql.cover.jpg"
---

Hace algunas semanas, un cliente me contactó con un problema muy común:

> "La búsqueda en la consola administrativa es lenta. A veces tarda
> varios segundos. Ya hemos reducido las JOIN, pero el problema no ha
> desaparecido."

Entorno: PostgreSQL en cloud managed.\
Tabla principal: `payment_report` (\~6 millones de filas, 3 GB).\
Columna buscada: `reference_code`.

Query problemática:

``` sql
SELECT *
FROM reporting.payment_report r
JOIN reporting.payment_cart c ON c.id = r.cart_id
WHERE c.service_id = 1001
  AND r.reference_code LIKE '%ABC123%'
ORDER BY c.created_at DESC
LIMIT 100;
```

------------------------------------------------------------------------

## 🧠 Primera observación: las JOIN no eran el problema

Comparé:

-   Versión AS-IS (3 JOIN sobre la misma tabla)
-   Versión TO-BE (solo 1 JOIN)

¿El resultado?

El plan de ejecución mostraba en ambos casos:

``` text
Parallel Seq Scan on payment_report
Rows Removed by Filter: ~2,000,000
Buffers: shared read = cientos de miles
Execution Time: 14–18 seconds
```

Reducir las JOIN tuvo un impacto marginal.

El verdadero problema era otro.

------------------------------------------------------------------------

## 📌 El culpable: `LIKE '%valor%'` sin un índice adecuado

Una búsqueda con wildcard inicial (`%valor%`) hace inutilizable un
índice {{< glossary term="b-tree" >}}B-Tree{{< /glossary >}} normal.

PostgreSQL se ve obligado a realizar un escaneo secuencial de toda la
tabla.

En este caso específico:

-   \~3 GB de datos
-   cientos de miles de páginas de 8KB leídas
-   carga dominada por I/O
-   segundos de latencia

No es un problema de "SQL mal escrito". Es un problema de access path.

------------------------------------------------------------------------

## 🔬 Antes de crear un índice: análisis de riesgo

El cliente preguntó con razón:

> "Si creamos un índice trigram (GIN), ¿corremos el riesgo de ralentizar
> las transacciones de pago?"

Aquí entra en juego un concepto que a menudo se ignora: el {{< glossary term="churn" >}}**churn**{{< /glossary >}}.

### ¿Qué es el churn?

Representa cuánto cambia una tabla después de insertar las filas.

Alta frecuencia de: - UPDATE - DELETE

→ alto churn\
→ mayor coste de mantenimiento del índice\
→ posible degradación en escrituras

En nuestro caso:

Tabla `payment_report`: - \~12k inserciones/día - 0 updates - 0
deletes - 0 dead tuples

Perfil: **append-only**

Este es el mejor escenario posible para introducir un índice GIN.

------------------------------------------------------------------------

## 📊 Verificación clave: ¿sincrónico o batch?

La tabla no tenía timestamp de inserción.

Solución: análisis indirecto.

Correlacioné las filas de `payment_report` con el timestamp del carrito
(`payment_cart.created_at`) y analicé la distribución horaria.

Resultado:

-   patrón continuo 24/7
-   picos diurnos
-   descenso nocturno
-   correlación perfecta con el tráfico de carritos

Conclusión: carga near real-time, no batch nocturno.

------------------------------------------------------------------------

## 🛠️ La solución

``` sql
CREATE EXTENSION IF NOT EXISTS {{< glossary term="pg-trgm" >}}pg_trgm{{< /glossary >}};

CREATE INDEX CONCURRENTLY idx_payment_report_reference_trgm
ON reporting.payment_report
USING {{< glossary term="gin-index" >}}gin{{< /glossary >}} (reference_code gin_trgm_ops);
```

Precauciones:

-   Crear en ventana off-peak
-   Usar modo CONCURRENTLY
-   Monitorizar I/O durante la creación

------------------------------------------------------------------------

## 📈 Resultado: el plan de ejecución antes y después

Este es el plan de ejecución completo de la query — antes y después de crear el índice trigram.

**Antes** (sin índice trigram):

``` text
Nested Loop Inner Join
  → Nested Loop Inner Join
    → Nested Loop Inner Join
      → Seq Scan on payment_report as r
          Filter: ((reference_code)::text ~~ '%ABC123%'::text)
      → Index Scan using payment_cart_pkey on payment_cart as c
          Filter: (service_id = 1001)
          Index Cond: (id = r.cart_id)
    → Index Only Scan using payment_cart_pkey on payment_cart as c2
        Index Cond: (id = c.id)
  → Index Only Scan using payment_cart_pkey on payment_cart as c3
      Index Cond: (id = c.id)
```

**Después** (con índice trigram):

``` text
Nested Loop Inner Join
  → Nested Loop Inner Join
    → Nested Loop Inner Join
      → Bitmap Heap Scan on payment_report as r
          Recheck Cond: ((reference_code)::text ~~ '%ABC123%'::text)
        → Bitmap Index Scan using idx_payment_report_reference_trgm
            Index Cond: ((reference_code)::text ~~ '%ABC123%'::text)
      → Index Scan using payment_cart_pkey on payment_cart as c
          Filter: (service_id = 1001)
          Index Cond: (id = r.cart_id)
    → Index Only Scan using payment_cart_pkey on payment_cart as c2
        Index Cond: (id = c.id)
  → Index Only Scan using payment_cart_pkey on payment_cart as c3
      Index Cond: (id = c.id)
```

El punto clave está en los pasos 4–5: el `Seq Scan` — que leía toda la tabla fila por fila — ha sido reemplazado por un `Bitmap Heap Scan` guiado por el índice trigram `idx_payment_report_reference_trgm`. PostgreSQL ahora filtra directamente a través del índice y solo hace el recheck en las filas candidatas.

Misma query, mismo dato, pero un access path completamente diferente. De segundos a milisegundos.

------------------------------------------------------------------------

## 🎯 Lección clave

Cuando una query es lenta:

1.  No te detengas en el número de JOIN.
2.  Mira el plan de ejecución.
3.  Identifica si el cuello de botella es CPU o I/O.
4.  Evalúa el churn antes de introducir un índice GIN.
5.  Mide siempre antes de decidir.

A menudo el problema no es "optimizar la query".\
Es darle al planner el índice correcto.

------------------------------------------------------------------------

## 💬 ¿Por qué comparto este caso?

Porque es un escenario extremadamente común:

-   Tablas grandes
-   Búsquedas tipo "contiene"
-   Miedo a introducir índices GIN
-   Temor a degradar el rendimiento de escritura

Con datos en la mano, la decisión se vuelve técnica, no emocional.

La optimización no es magia.\
Es medición, análisis de planes y comprensión del comportamiento real
del sistema.

------------------------------------------------------------------------

## Glosario

**[GIN Index](/es/glossary/gin-index/)** — Generalized Inverted Index: tipo de índice PostgreSQL que crea un mapping inverso de cada elemento a los registros que lo contienen. Ideal para búsquedas "contiene" en texto con pg_trgm.

**[B-Tree](/es/glossary/b-tree/)** — Estructura de datos de árbol balanceado, índice predeterminado en bases de datos relacionales. Eficiente para búsquedas de igualdad y rango, pero inutilizable para `LIKE '%valor%'`.

**[pg_trgm](/es/glossary/pg-trgm/)** — Extensión PostgreSQL que descompone texto en trigramas (secuencias de 3 caracteres), habilitando índices GIN para acelerar búsquedas con wildcard.

**[Churn](/es/glossary/churn/)** — Medida de cuánto cambia una tabla después de la inserción. Bajo churn (append-only) es el mejor escenario para introducir un índice GIN sin degradar las escrituras.

**[Execution Plan](/es/glossary/execution-plan/)** — Secuencia de operaciones elegida por la base de datos para resolver una query. Leer el plan es el primer paso para identificar si el problema es CPU, I/O o un access path incorrecto.
