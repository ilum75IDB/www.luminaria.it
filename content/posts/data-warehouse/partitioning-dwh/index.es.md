---
title: "Partitioning en el DWH: cuando 3 años de datos pesan demasiado"
description: "Una fact table de 800 millones de filas sin partitioning, queries trimestrales que tardaban 12 minutos y un negocio que quería respuestas en tiempo real. Cómo implementé el range partitioning mensual y llevé los tiempos a 40 segundos."
date: "2026-04-07T10:00:00+01:00"
draft: false
translationKey: "partitioning_dwh"
tags: ["partitioning", "performance", "oracle", "fact-table", "data-warehouse"]
categories: ["data-warehouse"]
image: "partitioning-dwh.cover.jpg"
---

La semana pasada un colega me contó sobre un proyecto donde las queries del data warehouse habían dejado de volver en tiempos razonables. "¿Cuánto tarda el informe trimestral?" le pregunté. "Doce minutos." "¿Y antes?" "Un minuto y medio."

No tuve que preguntar más. Ya conocía el guion.

Una {{< glossary term="fact-table" >}}fact table{{< /glossary >}} que empieza pequeña, crece cada día, y nadie se preocupa por la estructura física hasta que un día las queries dejan de volver. No es un bug, no es un error de código. Es el peso de los datos que al final se hace notar.

---

## El contexto: distribución y tres años de tickets de venta

El proyecto era en el sector de la gran distribución organizada — una cadena de supermercados con unos doscientos puntos de venta, alrededor de cien millones de euros de facturación anual, y un data warehouse Oracle 19c que recogía todo: ventas, devoluciones, movimientos de almacén, promociones.

La tabla en el centro del problema se llamaba `FACT_VENDITE`. Cada fila era una línea de ticket — un ticket promedio tiene ocho líneas, multiplicado por treinta mil tickets al día en doscientas tiendas, son unos 48 millones de filas al mes. En tres años se habían acumulado 800 millones de filas.

La estructura era esta:

```sql
CREATE TABLE fact_vendite (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20),
    CONSTRAINT pk_fact_vendite PRIMARY KEY (vendita_id)
);
```

Un solo índice en la primary key, un índice en `data_vendita` y uno compuesto en `(punto_vendita_id, data_vendita)`. Sin particionamiento. Ochocientos millones de filas en una única tabla monolítica.

## 🔍 El síntoma: full table scan sobre 800 millones de filas

Las queries analíticas del DWH trabajaban casi siempre por período. Ventas del último trimestre por punto de venta. Comparativa año contra año por categoría de producto. Márgenes mensuales por región. Todas con un filtro en `data_vendita`.

El informe trimestral era este:

```sql
SELECT pv.regione,
       cat.famiglia,
       SUM(f.importo)    AS fatturato,
       SUM(f.quantita)   AS pezzi_venduti,
       SUM(f.sconto)     AS sconto_totale
FROM   fact_vendite f
JOIN   dim_punto_vendita pv  ON f.punto_vendita_id = pv.punto_vendita_id
JOIN   dim_prodotto cat      ON f.prodotto_id      = cat.prodotto_id
WHERE  f.data_vendita BETWEEN DATE '2025-10-01' AND DATE '2025-12-31'
GROUP BY pv.regione, cat.famiglia
ORDER BY fatturato DESC;
```

El predicado sobre `data_vendita` debería haber usado el índice. Y lo hacía — un año antes, cuando la tabla tenía 500 millones de filas. Pero con 800 millones, el optimizer había decidido que el índice ya no convenía. El cálculo era sencillo: un trimestre = aproximadamente el 8% de las filas totales. Con un index range scan, Oracle habría necesitado 64 millones de accesos aleatorios a bloques. Un {{< glossary term="full-table-scan" >}}full table scan{{< /glossary >}} secuencial costaba menos.

Y eso hizo: leyó 800 millones de filas para devolver 64 millones.

```
--------------------------------------------------------------
| Id | Operation          | Name         | Rows  | Bytes    |
--------------------------------------------------------------
|  0 | SELECT STATEMENT   |              |  1200 |    84K   |
|  1 |  SORT GROUP BY     |              |  1200 |    84K   |
|* 2 |   HASH JOIN        |              |   64M | 4480M    |
|  3 |    TABLE ACCESS FULL| DIM_PRODOTTO |  12K  |  168K    |
|* 4 |    HASH JOIN        |              |   64M | 3520M    |
|  5 |     TABLE ACCESS FULL| DIM_PUNTO_VENDITA | 200 | 4000 |
|* 6 |     TABLE ACCESS FULL| FACT_VENDITE| 800M  | 40G      |
--------------------------------------------------------------
```

Cuarenta gigabytes de I/O para una query trimestral. En un entorno donde el buffer pool estaba dimensionado a 16 GB, significaba leer más de dos veces la caché completa desde disco. Doce minutos.

## 🏗️ La solución: range partitioning mensual

El particionamiento range por fecha es la elección natural para una fact table en un data warehouse. Los datos entran en orden cronológico, las queries filtran por período, los datos viejos se enfrían y los nuevos están calientes. La fecha es la clave de particionamiento perfecta.

Elegí el particionamiento mensual — 36 particiones para tres años de histórico, más una partición para los datos actuales. Cada partición contenía unos 48 millones de filas: un volumen manejable para las queries y para las operaciones de mantenimiento.

```sql
CREATE TABLE fact_vendite_part (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20)
)
PARTITION BY RANGE (data_vendita) (
    PARTITION p_2023_01 VALUES LESS THAN (DATE '2023-02-01'),
    PARTITION p_2023_02 VALUES LESS THAN (DATE '2023-03-01'),
    PARTITION p_2023_03 VALUES LESS THAN (DATE '2023-04-01'),
    -- ... 33 particiones intermedias ...
    PARTITION p_2025_12 VALUES LESS THAN (DATE '2026-01-01'),
    PARTITION p_max     VALUES LESS THAN (MAXVALUE)
);
```

Con un {{< glossary term="local-index" >}}índice local{{< /glossary >}} sobre la fecha:

```sql
CREATE INDEX idx_vendite_data_local ON fact_vendite_part (data_vendita) LOCAL;
CREATE INDEX idx_vendite_pv_local   ON fact_vendite_part (punto_vendita_id, data_vendita) LOCAL;
```

Cada partición tiene su propio segmento de índice. Cuando el optimizer elimina una partición, elimina también el segmento de índice correspondiente.

## 📦 La migración: de monolito a particiones

Migrar 800 millones de filas no es algo que se haga con un simple INSERT...SELECT. Hace falta una estrategia.

Usé el enfoque {{< glossary term="ctas" >}}CTAS{{< /glossary >}} (Create Table As Select) con {{< glossary term="nologging" >}}NOLOGGING{{< /glossary >}} y paralelismo. El procedimiento fue:

1. Crear la tabla particionada vacía con la estructura definitiva
2. Poblarla con un INSERT directo desde la tabla original
3. Reconstruir los índices
4. Validar los conteos
5. Renombrar las tablas (swap)
6. Ejecutar un backup RMAN inmediato (NOLOGGING lo requiere)

```sql
ALTER SESSION ENABLE PARALLEL DML;

INSERT /*+ APPEND PARALLEL(8) */ INTO fact_vendite_part
SELECT * FROM fact_vendite;

COMMIT;
```

Con 8 procesos paralelos y NOLOGGING, la carga tomó 47 minutos para 800 millones de filas. Nada mal, considerando que cada fila tenía que ser distribuida en la partición correcta según su fecha.

Luego la fase de validación:

```sql
SELECT 'Original' AS fuente, COUNT(*) AS filas FROM fact_vendite
UNION ALL
SELECT 'Particionada', COUNT(*) FROM fact_vendite_part;
```

800.247.331 filas en ambos lados. Perfecto.

```sql
ALTER TABLE fact_vendite RENAME TO fact_vendite_old;
ALTER TABLE fact_vendite_part RENAME TO fact_vendite;
```

La tabla original la mantuve una semana como red de seguridad, luego la eliminé.

## ⚡ El {{< glossary term="partition-pruning" >}}partition pruning{{< /glossary >}} en acción

Con el particionamiento en su lugar, la misma query trimestral cambiaba completamente de plan de ejecución:

```sql
SELECT pv.regione,
       cat.famiglia,
       SUM(f.importo)    AS fatturato,
       SUM(f.quantita)   AS pezzi_venduti,
       SUM(f.sconto)     AS sconto_totale
FROM   fact_vendite f
JOIN   dim_punto_vendita pv  ON f.punto_vendita_id = pv.punto_vendita_id
JOIN   dim_prodotto cat      ON f.prodotto_id      = cat.prodotto_id
WHERE  f.data_vendita BETWEEN DATE '2025-10-01' AND DATE '2025-12-31'
GROUP BY pv.regione, cat.famiglia
ORDER BY fatturato DESC;
```

```
----------------------------------------------------------------------
| Id | Operation                   | Name         | Pstart | Pstop  |
----------------------------------------------------------------------
|  0 | SELECT STATEMENT            |              |        |        |
|  1 |  SORT GROUP BY              |              |        |        |
|* 2 |   HASH JOIN                 |              |        |        |
|  3 |    TABLE ACCESS FULL        | DIM_PRODOTTO |        |        |
|* 4 |    HASH JOIN                |              |        |        |
|  5 |     TABLE ACCESS FULL       | DIM_PUNTO_V  |        |        |
|  6 |     PARTITION RANGE ITERATOR|              |  34    |  36    |
|* 7 |      TABLE ACCESS FULL      | FACT_VENDITE |  34    |  36    |
----------------------------------------------------------------------
```

`Pstart: 34, Pstop: 36`. El optimizer leyó solo tres particiones de 37 — octubre, noviembre y diciembre 2025. En vez de 800 millones de filas, escaneó 144 millones. En vez de 40 GB de I/O, unos 7 GB.

¿El resultado? De 12 minutos a 40 segundos.

No porque el hardware fuera más rápido, no porque hubiera reescrito las queries. Solo porque la base de datos ahora sabía dónde *no* buscar.

## 🔄 Exchange partition: la carga que no cuesta nada

En un data warehouse, los datos llegan con una cadencia regular — en nuestro caso, un {{< glossary term="etl" >}}ETL{{< /glossary >}} nocturno que cargaba las ventas del día. El problema clásico del particionamiento es: ¿cómo cargas los nuevos datos en la partición correcta sin impactar las queries?

La respuesta se llama {{< glossary term="exchange-partition" >}}exchange partition{{< /glossary >}}.

El proceso funcionaba así:

1. El ETL carga los datos del día en una tabla de staging sin particionar
2. Se construyen los índices sobre la staging table (misma estructura que los índices locales)
3. Se ejecuta el exchange partition: la staging table y la partición objetivo intercambian sus segmentos

```sql
-- 1. Tabla de staging con misma estructura que una partición
CREATE TABLE stg_vendite_daily (
    vendita_id       NUMBER(18)    NOT NULL,
    data_vendita     DATE          NOT NULL,
    punto_vendita_id NUMBER(10)    NOT NULL,
    prodotto_id      NUMBER(10)    NOT NULL,
    cliente_id       NUMBER(10),
    quantita         NUMBER(10,2)  NOT NULL,
    importo          NUMBER(12,2)  NOT NULL,
    sconto           NUMBER(8,2)   DEFAULT 0,
    tipo_pagamento   VARCHAR2(20)
);

-- 2. Carga ETL en staging (operación independiente)
INSERT /*+ APPEND */ INTO stg_vendite_daily
SELECT * FROM source_vendite WHERE data_vendita = TRUNC(SYSDATE - 1);

-- 3. Exchange: swap instantáneo de segmentos
ALTER TABLE fact_vendite
EXCHANGE PARTITION p_2026_01 WITH TABLE stg_vendite_daily
INCLUDING INDEXES WITHOUT VALIDATION;
```

El exchange partition es una operación DDL que solo modifica el data dictionary — no mueve ni un solo byte de datos. Tarda menos de un segundo, independientemente del volumen. Y durante el exchange, las queries sobre las demás particiones siguen funcionando sin interrupción.

En nuestro caso, el ETL nocturno acumulaba los datos del día en la staging, y a fin de mes se hacía el exchange con la partición del mes corriente. Durante el mes, los datos diarios iban a la partición `p_max` (la catch-all) y luego se consolidaban con un exchange mensual.

## 📊 El ciclo de vida de los datos

Con el particionamiento, la gestión del ciclo de vida se vuelve trivial. Después de tres años, la partición más antigua se puede:

- **comprimir**: `ALTER TABLE fact_vendite MODIFY PARTITION p_2023_01 COMPRESS FOR QUERY HIGH;`
- **mover a almacenamiento más lento**: `ALTER TABLE fact_vendite MOVE PARTITION p_2023_01 TABLESPACE ts_archivio;`
- **eliminar directamente**: `ALTER TABLE fact_vendite DROP PARTITION p_2023_01;`

Eliminar una partición es instantáneo — es una operación sobre el data dictionary, no borra fila por fila. Compáralo con un `DELETE FROM fact_vendite WHERE data_vendita < DATE '2023-02-01'` sobre 48 millones de filas: minutos de procesamiento, toneladas de redo log, y una tabla llena de espacio recuperable que necesita un reorganize.

En el proyecto de distribución, la política era: 3 años online comprimidos, luego drop. Cada primero de mes un job programado creaba la nueva partición y, si era necesario, eliminaba la de 37 meses antes. Completamente automático.

## 🎯 Lo que el particionamiento no resuelve

El particionamiento no es una varita mágica. No sustituye a los índices — si la query no filtra por la clave de particionamiento, el pruning no se activa y la base de datos lee todas las particiones. No mejora las queries que ya usan un índice eficiente sobre pocas filas. Y añade complejidad en la gestión: particiones por crear, monitorear, comprimir, eliminar.

Pero para una fact table en un data warehouse — donde los datos son cronológicos, las queries filtran por período, y los volúmenes crecen cada día — el particionamiento range por fecha no es una opción. Es un requisito arquitectural.

El colega con el informe de 12 minutos no tenía un problema de hardware ni de queries mal escritas. Tenía una tabla que había crecido más allá del punto donde la falta de estructura física se convierte en cuello de botella. El particionamiento puso las cosas en su lugar: 40 segundos, y ninguna fila leída innecesariamente.

------------------------------------------------------------------------

## Glosario

**[Range Partitioning](/es/glossary/range-partitioning/)** — Estrategia de particionamiento que divide una tabla en segmentos basados en rangos de valores de una columna (típicamente una fecha). Cada partición contiene las filas cuyo valor cae dentro del rango definido.

**[Exchange Partition](/es/glossary/exchange-partition/)** — Operación DDL de Oracle que intercambia instantáneamente los segmentos de datos entre una tabla no particionada y una partición, sin mover físicamente ningún dato. Usada en data warehouses para cargas masivas sin impacto.

**[Partition Pruning](/es/glossary/partition-pruning/)** — Mecanismo automático del optimizer Oracle que excluye las particiones no relevantes durante la ejecución de una query, leyendo solo las que corresponden al predicado WHERE.

**[Fact table](/es/glossary/fact-table/)** — Tabla central del star schema que contiene las medidas numéricas del negocio (importes, cantidades, conteos) y las claves foráneas hacia las tablas dimensionales.

**[Full Table Scan](/es/glossary/full-table-scan/)** — Operación de lectura en la que la base de datos recorre todos los bloques de una tabla sin utilizar índices. Eficiente sobre grandes volúmenes cuando la selectividad es baja, costosa cuando se buscan pocas filas.
