---
title: "SCD Tipo 2: la historia que el negocio no sabía que necesitaba"
description: "Un director comercial pregunta cuántos clientes tenía la región Norte en junio pasado. El DWH no sabe responder porque cada actualización sobrescribe los datos anteriores. Cómo implementé una SCD Tipo 2 con claves subrogadas y fechas de validez para devolver al negocio su memoria histórica."
date: "2025-11-11T10:00:00+01:00"
draft: false
translationKey: "scd_tipo_2"
tags: ["scd", "dimensional-modeling", "etl", "kimball", "data-warehouse"]
categories: ["data-warehouse"]
image: "scd-tipo-2.cover.jpg"
---

El director comercial se presenta en la reunión del lunes por la mañana con una pregunta sencilla: "¿Cuántos clientes teníamos en la región Norte en junio pasado?"

Respuesta del DWH: silencio.

No porque el sistema estuviera caído, ni porque faltara la tabla. El dato estaba ahí, técnicamente. Pero era erróneo. El DWH devolvía los clientes que *hoy* están en la región Norte — no los que estaban en junio. Porque cada noche, el proceso de carga sobrescribía la tabla maestra de clientes con los valores actuales, borrando cualquier rastro de lo que había antes.

Un cliente que en junio estaba en la región Norte y en septiembre se trasladó a la región Centro? Para el DWH, ese cliente siempre había estado en la región Centro. La historia no existía.

---

## El proyecto y el modelo original

El contexto era un data warehouse en el sector asegurador — gestión de siniestros y cartera de clientes. El sistema fuente contenía un registro maestro por cada cliente: nombre, región, agente asignado, clase de riesgo, tipo de póliza.

La dimensión en el DWH estaba modelada así:

``` sql
CREATE TABLE dim_cliente (
    cliente_id      NUMBER(10)    NOT NULL,
    nombre          VARCHAR2(100) NOT NULL,
    region          VARCHAR2(50)  NOT NULL,
    agente          VARCHAR2(100),
    clase_riesgo    VARCHAR2(20),
    tipo_poliza     VARCHAR2(50),
    CONSTRAINT pk_dim_cliente PRIMARY KEY (cliente_id)
);
```

El ETL nocturno era un simple MERGE: si el cliente existe, actualiza todos los campos; si no existe, inserta.

``` sql
MERGE INTO dim_cliente d
USING stg_cliente s ON (d.cliente_id = s.cliente_id)
WHEN MATCHED THEN UPDATE SET
    d.nombre       = s.nombre,
    d.region       = s.region,
    d.agente       = s.agente,
    d.clase_riesgo = s.clase_riesgo,
    d.tipo_poliza  = s.tipo_poliza
WHEN NOT MATCHED THEN INSERT (
    cliente_id, nombre, region, agente, clase_riesgo, tipo_poliza
) VALUES (
    s.cliente_id, s.nombre, s.region, s.agente, s.clase_riesgo, s.tipo_poliza
);
```

Simple, limpio, rápido. Y completamente equivocado para un data warehouse.

Esto es lo que Kimball llama **SCD Tipo 1** — Slowly Changing Dimension de Tipo 1. Sobrescribes el valor antiguo con el nuevo. Sin historia, sin versionado. El valor actual borra el anterior.

Para un sistema OLTP es perfecto: siempre quieres la dirección actual del cliente, el teléfono actualizado, el email válido. Pero un data warehouse no es un sistema transaccional. Un data warehouse es una máquina del tiempo. Y una máquina del tiempo que sobrescribe el pasado es inútil.

---

## Lo que se pierde con el Tipo 1

El director comercial no era el único que hacía preguntas que el DWH no podía responder. Aquí va una muestra de las solicitudes que se acumularon en tres meses:

- *"¿Cuántos clientes pasaron de la clase de riesgo Alta a Baja en el último año?"* — Imposible. La clase anterior ya no existe.
- *"¿El agente Rossi ha perdido clientes respecto al trimestre pasado?"* — Imposible. Si un cliente fue reasignado al agente Bianchi, no queda rastro de que alguna vez perteneció a Rossi.
- *"¿La facturación de la región Sur bajó o los clientes se mudaron?"* — Imposible de distinguir. Si un cliente de 200K se trasladó de la región Sur a la Centro, la facturación del Sur baja pero no porque el negocio vaya mal — simplemente el cliente cambió de dirección.

Cada vez la respuesta era la misma: "El sistema no guarda historia." Que traducido al lenguaje empresarial significa: "No lo sabemos."

En un momento dado, el CFO pidió un informe de análisis trimestral comparando la composición de la cartera de clientes entre Q1 y Q2. El equipo de BI intentó construirlo. Tardaron tres días. El resultado no era fiable porque los datos de Q1 ya no existían — habían sido sobrescritos con los de Q2. El informe comparaba Q2 con Q2 disfrazado de Q1.

Ese fue el momento que desencadenó el proyecto de reestructuración.

---

## SCD Tipo 2: el principio

El Tipo 2 no sobrescribe. Versiona.

Cuando un atributo cambia, el registro actual se cierra — se le asigna una fecha de fin de validez — y se inserta un nuevo registro con los valores actualizados y una nueva fecha de inicio de validez. El registro antiguo permanece en la base de datos, intacto, con todos los valores que tenía cuando era el registro vigente.

Para que esto funcione se necesitan tres elementos adicionales en la tabla dimensional:

1. **Una clave subrogada** — un identificador generado por el DWH, distinto de la clave natural del sistema fuente. Es necesaria porque el mismo cliente tendrá múltiples registros (uno por cada versión), así que la clave natural ya no es única.
2. **Fechas de validez** — `valid_from` y `valid_to` — que definen el intervalo temporal en que cada versión del registro era la vigente.
3. **Un flag de versión actual** — `is_current` — que permite recuperar rápidamente la versión activa sin filtrar por fechas.

### La nueva tabla dimensional

``` sql
CREATE TABLE dim_cliente (
    cliente_key     NUMBER(10)    NOT NULL,
    cliente_id      NUMBER(10)    NOT NULL,
    nombre          VARCHAR2(100) NOT NULL,
    region          VARCHAR2(50)  NOT NULL,
    agente          VARCHAR2(100),
    clase_riesgo    VARCHAR2(20),
    tipo_poliza     VARCHAR2(50),
    valid_from      DATE          NOT NULL,
    valid_to        DATE          NOT NULL,
    is_current      CHAR(1)       DEFAULT 'Y' NOT NULL,
    CONSTRAINT pk_dim_cliente PRIMARY KEY (cliente_key)
);

CREATE INDEX idx_dim_cliente_natural ON dim_cliente (cliente_id, is_current);
CREATE INDEX idx_dim_cliente_validity ON dim_cliente (cliente_id, valid_from, valid_to);

CREATE SEQUENCE seq_dim_cliente START WITH 1 INCREMENT BY 1;
```

La `cliente_key` es la clave subrogada — generada por la secuencia, nunca tomada del sistema fuente. La `cliente_id` es la clave natural — sirve para vincular las diferentes versiones del mismo cliente.

La `valid_to` para el registro actual la fijo en `DATE '9999-12-31'` — una convención estándar que simplifica las consultas temporales. Cuando buscas el registro válido en una fecha determinada, el filtro `WHERE fecha_referencia BETWEEN valid_from AND valid_to` funciona sin casos especiales.

---

## La lógica ETL

El ETL del Tipo 2 tiene dos fases: primero cierra los registros que han cambiado, luego inserta las nuevas versiones. El orden importa — si insertas antes de cerrar, hay un momento en que existen dos versiones "actuales" del mismo cliente.

### Fase 1: identificar y cerrar los registros modificados

``` sql
MERGE INTO dim_cliente d
USING (
    SELECT s.cliente_id,
           s.nombre,
           s.region,
           s.agente,
           s.clase_riesgo,
           s.tipo_poliza
    FROM   stg_cliente s
    JOIN   dim_cliente d
           ON  s.cliente_id = d.cliente_id
           AND d.is_current = 'Y'
    WHERE  (s.region       != d.region
         OR s.agente       != d.agente
         OR s.clase_riesgo != d.clase_riesgo
         OR s.tipo_poliza  != d.tipo_poliza
         OR s.nombre       != d.nombre)
) changed ON (d.cliente_id = changed.cliente_id AND d.is_current = 'Y')
WHEN MATCHED THEN UPDATE SET
    d.valid_to   = TRUNC(SYSDATE) - 1,
    d.is_current = 'N';
```

El WHERE compara cada atributo rastreado. Si al menos uno es diferente, el registro actual se cierra: la `valid_to` se establece a ayer y `is_current` pasa a 'N'.

Una nota práctica: la comparación con `!=` no gestiona los NULL. Si `agente` puede ser NULL, se necesitan funciones de comparación NULL-safe. En Oracle uso `DECODE`:

``` sql
WHERE DECODE(s.region, d.region, 0, 1) = 1
   OR DECODE(s.agente, d.agente, 0, 1) = 1
   OR DECODE(s.clase_riesgo, d.clase_riesgo, 0, 1) = 1
   -- ...
```

`DECODE` trata dos NULL como iguales — exactamente el comportamiento que necesitas.

### Fase 2: insertar las nuevas versiones

``` sql
INSERT INTO dim_cliente (
    cliente_key, cliente_id, nombre, region, agente,
    clase_riesgo, tipo_poliza, valid_from, valid_to, is_current
)
SELECT seq_dim_cliente.NEXTVAL,
       s.cliente_id,
       s.nombre,
       s.region,
       s.agente,
       s.clase_riesgo,
       s.tipo_poliza,
       TRUNC(SYSDATE),
       DATE '9999-12-31',
       'Y'
FROM   stg_cliente s
WHERE  NOT EXISTS (
    SELECT 1
    FROM   dim_cliente d
    WHERE  d.cliente_id = s.cliente_id
    AND    d.is_current = 'Y'
);
```

Este INSERT captura dos casos: clientes completamente nuevos (que no existen en dim_cliente) y clientes cuya versión actual acaba de ser cerrada en la Fase 1 (que por lo tanto ya no tienen un registro con `is_current = 'Y'`).

La `valid_from` es la fecha de hoy. La `valid_to` es el "fin de los tiempos" — `9999-12-31`. La `cliente_key` la genera la secuencia.

---

## Los datos: antes y después

Veamos un ejemplo concreto. El cliente 2001 — "Alfa Seguros Srl" — está en la región Norte, asignado al agente Rossi, clase de riesgo Media.

En julio el cliente es reasignado al agente Bianchi. En octubre cambia de clase de riesgo de Media a Alta.

**Con el Tipo 1** (el modelo anterior), en octubre dim_cliente contiene una sola fila:

``` text
CLIENTE_ID  NOMBRE               REGION  AGENTE   CLASE_RIESGO
----------  -------------------  ------  -------  ------------
2001        Alfa Seguros Srl     Norte   Bianchi  Alta
```

Ningún rastro de Rossi. Ningún rastro de la clase Media. Para el DWH, este cliente siempre ha pertenecido al agente Bianchi con clase Alta.

**Con el Tipo 2**, en octubre dim_cliente contiene tres filas:

``` text
KEY   CLIENTE_ID  NOMBRE               REGION  AGENTE   CLASE   VALID_FROM  VALID_TO    CURRENT
----  ----------  -------------------  ------  -------  ------  ----------  ----------  -------
1001  2001        Alfa Seguros Srl     Norte   Rossi    Media   2025-01-15  2025-07-09  N
1002  2001        Alfa Seguros Srl     Norte   Bianchi  Media   2025-07-10  2025-10-04  N
1003  2001        Alfa Seguros Srl     Norte   Bianchi  Alta    2025-10-05  9999-12-31  Y
```

Tres versiones del mismo cliente. Cada versión cuenta un trozo de la historia: quién era el agente, cuál era la clase de riesgo, y en qué período. Las fechas no se solapan. El flag `is_current` identifica la versión activa.

---

## Las consultas temporales

Ahora el director comercial puede tener su respuesta.

### ¿Cuántos clientes en la región Norte en junio?

``` sql
SELECT COUNT(DISTINCT cliente_id) AS clientes_norte_junio
FROM   dim_cliente
WHERE  region = 'Norte'
AND    DATE '2025-06-15' BETWEEN valid_from AND valid_to;
```

La consulta es directa: toma todos los registros que eran válidos el 15 de junio de 2025 y filtra por región. Sin CASE WHEN, sin lógica condicional, sin aproximaciones.

### Clientes que cambiaron de clase de riesgo en el último año

``` sql
SELECT c1.cliente_id,
       c1.nombre,
       c1.clase_riesgo  AS clase_anterior,
       c2.clase_riesgo  AS clase_actual,
       c1.valid_to + 1  AS fecha_cambio
FROM   dim_cliente c1
JOIN   dim_cliente c2
       ON  c1.cliente_id = c2.cliente_id
       AND c1.valid_to + 1 = c2.valid_from
WHERE  c1.clase_riesgo != c2.clase_riesgo
AND    c1.valid_to >= ADD_MONTHS(TRUNC(SYSDATE), -12)
ORDER BY fecha_cambio DESC;
```

Dos versiones consecutivas del mismo cliente, unidas por fecha de transición. Si la clase de riesgo difiere entre las dos versiones, el cliente ha cambiado de clase. La fecha de cambio es el día posterior al cierre de la versión anterior.

### Comparación de cartera Q1 vs Q2

``` sql
SELECT region,
       COUNT(DISTINCT CASE
           WHEN DATE '2025-03-31' BETWEEN valid_from AND valid_to
           THEN cliente_id END) AS clientes_q1,
       COUNT(DISTINCT CASE
           WHEN DATE '2025-06-30' BETWEEN valid_from AND valid_to
           THEN cliente_id END) AS clientes_q2
FROM   dim_cliente
WHERE  DATE '2025-03-31' BETWEEN valid_from AND valid_to
   OR  DATE '2025-06-30' BETWEEN valid_from AND valid_to
GROUP BY region
ORDER BY region;
```

Un solo scan de la tabla, dos conteos distintos filtrados por fecha. El CFO tiene su informe trimestral — el real, no el que comparaba Q2 consigo mismo.

---

## La tabla de hechos y las claves subrogadas

Un punto que a menudo se subestima: la tabla de hechos debe usar la **clave subrogada**, no la clave natural.

``` sql
CREATE TABLE fact_siniestro (
    siniestro_key   NUMBER(10)    NOT NULL,
    cliente_key     NUMBER(10)    NOT NULL,  -- FK a la versión específica
    fecha_key       NUMBER(8)     NOT NULL,
    importe         NUMBER(15,2),
    tipo_siniestro  VARCHAR2(50),
    CONSTRAINT pk_fact_siniestro PRIMARY KEY (siniestro_key),
    CONSTRAINT fk_fact_cliente FOREIGN KEY (cliente_key)
        REFERENCES dim_cliente (cliente_key)
);
```

La `cliente_key` en la tabla de hechos apunta a la *versión* del cliente que era vigente en el momento del siniestro. Si un siniestro ocurre en mayo, cuando el cliente todavía pertenecía al agente Rossi, el hecho apunta a la versión con agente Rossi. Si otro siniestro ocurre en septiembre, con el cliente ya bajo el agente Bianchi, el hecho apunta a la versión con agente Bianchi.

El resultado es que cada hecho está asociado al contexto dimensional correcto para el momento en que ocurrió. Consultas los siniestros de mayo y ves al agente Rossi. Consultas los de septiembre y ves al agente Bianchi. Sin ninguna lógica temporal en la consulta — el JOIN directo entre tabla de hechos y dimensión devuelve el contexto correcto.

``` sql
-- Siniestros por agente, con el contexto correcto en el momento del siniestro
SELECT d.agente,
       COUNT(*)       AS num_siniestros,
       SUM(f.importe) AS importe_total
FROM   fact_siniestro f
JOIN   dim_cliente d ON f.cliente_key = d.cliente_key
GROUP BY d.agente
ORDER BY importe_total DESC;
```

Sin cláusula temporal. El JOIN por clave subrogada hace todo el trabajo.

---

## Las dimensiones del Tipo 2

El coste del Tipo 2 es el crecimiento de la tabla dimensional. Con el Tipo 1, cada cliente es una fila. Con el Tipo 2, cada cliente puede tener N filas — una por cada cambio de atributo rastreado.

En el proyecto asegurador los números eran estos:

| Métrica | Valor |
|---------|-------|
| Clientes activos | ~120.000 |
| Atributos rastreados | 4 (región, agente, clase riesgo, tipo póliza) |
| Tasa de cambio media | ~8% de clientes/año |
| Filas dim_cliente tras 1 año | ~140.000 |
| Filas dim_cliente tras 3 años | ~180.000 |
| Filas dim_cliente tras 5 años | ~220.000 |

De 120K a 220K en cinco años. Un aumento del 83% — que parece mucho en porcentaje pero es despreciable en términos absolutos. 220K filas son nada para Oracle. La consulta con índice sobre la clave subrogada se mantiene en el orden de los milisegundos.

El problema se presenta cuando tienes millones de clientes con altas tasas de cambio. En ese caso monitorizas el crecimiento, consideras el particionamiento de la dimensión, y sobre todo eliges con cuidado *qué* atributos rastrear. No todos los atributos merecen Tipo 2. ¿El teléfono del cliente? Tipo 1, sobrescritura. ¿La región comercial? Tipo 2, porque impacta en el análisis de facturación.

La elección de qué atributos rastrear con Tipo 2 es una decisión de negocio, no técnica. Pregunta al negocio: "Si este campo cambia, ¿necesitáis saber cuál era el valor anterior?" Si la respuesta es sí, es Tipo 2. Si es no, es Tipo 1.

---

## Cuándo no hace falta el Tipo 2

No todas las dimensiones necesitan historia. He visto proyectos donde cada dimensión era Tipo 2 "por si acaso" — el resultado era un modelo innecesariamente complejo, ETL lentos, y nadie que jamás consultara la historia de la dimensión "tipo_pago" o "canal_venta".

El Tipo 2 tiene un coste: complejidad del ETL, crecimiento de la tabla, necesidad de gestionar claves subrogadas en la tabla de hechos. Es un coste que vale la pena pagar cuando el negocio necesita la historia. Si no la necesita, el Tipo 1 es la elección correcta.

También hay casos en que el Tipo 2 no es suficiente. Si necesitas saber no solo *qué* cambió sino también *quién* hizo el cambio y *por qué*, entonces necesitas un audit trail — una tabla separada con un log completo de modificaciones. El Tipo 2 rastrea versiones, no causas.

Y para dimensiones con cambios muy frecuentes — precios que cambian cada día, puntuaciones que se actualizan cada hora — el Tipo 2 puede generar un crecimiento insostenible. En esos casos se valora el **Tipo 6** (una combinación de Tipos 1, 2 y 3) o enfoques de mini-dimensión.

Pero para el caso más común — datos maestros de clientes, productos, empleados, puntos de venta — el Tipo 2 es la herramienta adecuada. Lo suficientemente simple para implementarse sin frameworks exóticos, lo suficientemente potente para devolver al negocio la dimensión que le faltaba: el tiempo.

---

## Lo que aprendí

El director comercial no sabía que necesitaba la historia hasta que la necesitó. Y cuando la necesitó, el DWH no la tenía.

Ese es el punto. No se implementa el Tipo 2 porque "es best practice" o porque "Kimball lo dice en el capítulo 5". Se implementa porque un data warehouse sin historia es una base de datos operativa con un star schema pegado encima. Funciona para los informes del mes actual, pero no responde a la pregunta que tarde o temprano alguien hará: "¿Cómo era antes?"

La pregunta siempre llega. La cuestión es si tu DWH está preparado para responder.
