---
title: "Oracle Partitioning: cuando 2 mil millones de filas ya no caben en una query"
description: "Un cliente con una tabla de transacciones de 2 mil millones de filas y consultas de reportes que habían pasado de segundos a horas. Cómo lo resolví con partitioning Oracle — range, interval, partition pruning e índices locales."
date: "2025-12-23T10:00:00+01:00"
draft: false
translationKey: "oracle_partitioning"
tags: ["partitioning", "performance", "tuning", "execution-plan"]
categories: ["oracle"]
image: "oracle-partitioning.cover.jpg"
---

Dos mil millones de filas. No es un número que se alcance en un día. Se necesitan años de transacciones, movimientos, registros diarios que se acumulan. Y durante todo ese tiempo la base de datos funciona, las consultas responden, los reportes salen. Luego un día alguien abre un ticket: "el reporte mensual tarda cuatro horas."

Cuatro horas. Para un reporte que seis meses antes tardaba veinte minutos.

No es un bug. No es un problema de red o de almacenamiento lento. Es la física de los datos: cuando una tabla crece más allá de cierto umbral, los enfoques que funcionaban dejan de funcionar. Y si no diseñaste la estructura para manejar ese crecimiento, la base de datos hace lo único que puede: leerlo todo.

---

## El contexto: telecomunicaciones y volúmenes industriales

El cliente era un operador de telecomunicaciones. Nada exótico — un clásico entorno Oracle 19c Enterprise Edition sobre Linux, almacenamiento SAN, unas treinta instancias entre producción, staging y desarrollo. La instancia crítica era la de facturación: facturación, CDR (Call Detail Records), movimientos contables.

La tabla en el centro del problema se llamaba `TXN_MOVIMENTI`. Recopilaba cada transacción individual del sistema de facturación desde 2016. La estructura era más o menos esta:

``` sql
CREATE TABLE txn_movimenti (
    txn_id         NUMBER(18)     NOT NULL,
    data_movimento DATE           NOT NULL,
    cod_cliente    VARCHAR2(20)   NOT NULL,
    tipo_movimento VARCHAR2(10)   NOT NULL,
    importo        NUMBER(15,4),
    canale         VARCHAR2(30),
    stato          VARCHAR2(5)    DEFAULT 'ATT',
    data_insert    TIMESTAMP      DEFAULT SYSTIMESTAMP,
    CONSTRAINT pk_txn_movimenti PRIMARY KEY (txn_id)
);
```

2.1 mil millones de filas. 380 GB de datos. Un solo segmento, un solo tablespace, sin particiones. Un monolito.

Los índices estaban ahí: uno en la clave primaria, uno en `data_movimento`, uno compuesto en `(cod_cliente, data_movimento)`. Pero cuando una tabla supera cierto tamaño, incluso un index range scan ya no es suficiente, porque el volumen de datos devuelto sigue siendo enorme.

---

## Los síntomas: no es lentitud, es colapso

Los problemas no aparecieron todos a la vez. Llegaron gradualmente, como sucede siempre con las tablas que crecen sin control.

**Primera señal**: los reportes mensuales. La consulta agregada de facturación — que sumaba importes por cliente para un mes dado — había pasado de 20 minutos a 4 horas en el transcurso de un año. El plan de ejecución mostraba un index range scan sobre la fecha, pero el número de bloques leídos era monstruoso: Oracle tenía que recorrer cientos de miles de leaf blocks del índice y luego hacer table access by rowid para recuperar las columnas no cubiertas.

**Segunda señal**: el mantenimiento. El `ALTER INDEX REBUILD` sobre el índice de la fecha requería seis horas. La recopilación de estadísticas (`DBMS_STATS.GATHER_TABLE_STATS`) no terminaba en una noche. Los backups RMAN se habían convertido en una ruleta: a veces entraban en la ventana, a veces no.

**Tercera señal**: los full table scan involuntarios. Consultas con predicados sobre la fecha que el optimizador decidía resolver con un full table scan porque el coste estimado del index scan era superior. Sobre 380 GB de datos.

El plan de ejecución de la consulta de facturación era este:

``` sql
SELECT cod_cliente,
       TRUNC(data_movimento, 'MM') AS mese,
       SUM(importo) AS totale
FROM   txn_movimenti
WHERE  data_movimento BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'
AND    stato = 'CON'
GROUP BY cod_cliente, TRUNC(data_movimento, 'MM');
```

``` text
---------------------------------------------------------------------
| Id  | Operation                    | Name            | Rows  | Cost |
---------------------------------------------------------------------
|   0 | SELECT STATEMENT             |                 |  125K | 890K |
|   1 |  HASH GROUP BY               |                 |  125K | 890K |
|   2 |   TABLE ACCESS BY INDEX ROWID| TXN_MOVIMENTI   |  28M  | 885K |
|*  3 |    INDEX RANGE SCAN          | IDX_TXN_DATA    |  28M  | 85K  |
---------------------------------------------------------------------
```

28 millones de filas solo para enero. El índice encontraba las filas, pero luego Oracle tenía que ir a buscar cada fila individual de la tabla para leer `cod_cliente`, `importo` y `stato`. Millones de operaciones de I/O aleatorio sobre una tabla de 380 GB dispersa en miles de bloques.

---

## La solución: no se necesita un índice mejor, se necesita una estructura diferente

Pasé dos días analizando los patrones de acceso antes de proponer cualquier solución. Porque el partitioning no es una varita mágica — si te equivocas con la clave de partición, empeoras las cosas.

Los patrones eran claros:

- El **90% de las consultas** tenía un predicado sobre la fecha (`data_movimento`)
- Los reportes eran siempre **mensuales o trimestrales**
- Las consultas operativas (cliente individual) usaban siempre `cod_cliente + data_movimento`
- Los datos de más de 3 años no se leían nunca en los reportes, solo en los procesos anuales de archivo

La elección recayó en un **interval partitioning mensual** sobre la columna `data_movimento`. No range partitioning clásico, donde hay que crear manualmente cada partición futura. Interval: defines el intervalo una vez y Oracle crea las particiones automáticamente cuando llegan datos para un nuevo período.

---

## La implementación: CTAS, índices locales y cero downtime (casi)

No se puede hacer `ALTER TABLE ... PARTITION BY` sobre una tabla existente con 2 mil millones de filas. No en Oracle 19c, al menos no sin Online Table Redefinition. Y esa opción, sobre una tabla de estas dimensiones, tiene sus propios riesgos.

Elegí el enfoque CTAS — Create Table As Select — con paralelismo. Crear la nueva tabla particionada, copiar los datos, renombrar.

### Paso 1: crear la tabla particionada

``` sql
CREATE TABLE txn_movimenti_part
PARTITION BY RANGE (data_movimento)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
    PARTITION p_before_2016 VALUES LESS THAN (DATE '2016-01-01'),
    PARTITION p_2016_01     VALUES LESS THAN (DATE '2016-02-01'),
    PARTITION p_2016_02     VALUES LESS THAN (DATE '2016-03-01')
    -- Oracle creará automáticamente las particiones posteriores
)
TABLESPACE ts_billing_data
NOLOGGING
PARALLEL 8
AS
SELECT /*+ PARALLEL(t, 8) */
       txn_id, data_movimento, cod_cliente, tipo_movimento,
       importo, canale, stato, data_insert
FROM   txn_movimenti t;
```

El `NOLOGGING` es fundamental: sin él, la copia genera redo log por cada bloque escrito. Sobre 380 GB significaría llenar el área de redo y poner el sistema en modo archivelog durante días. Con `NOLOGGING` la copia tardó 3 horas y media con paralelismo a 8.

Después de la copia restauré el logging:

``` sql
ALTER TABLE txn_movimenti_part LOGGING;
```

Y lancé un backup RMAN inmediatamente, porque los segmentos NOLOGGING no son recuperables en caso de restore.

### Paso 2: índices locales

El diseño de índices sobre una tabla particionada es diferente de una tabla normal. El concepto clave es: **índice local vs índice global**.

Un índice **local** está particionado con la misma clave que la tabla. Cada partición de la tabla tiene su partición de índice correspondiente. Ventaja: las operaciones de mantenimiento sobre una partición no tocan las demás.

Un índice **global** abarca todas las particiones. Es más eficiente para consultas que no filtran por la clave de partición, pero cualquier operación DDL sobre la partición (drop, truncate, split) invalida el índice entero.

``` sql
-- Clave primaria como índice global (necesario para búsquedas puntuales)
ALTER TABLE txn_movimenti_part
ADD CONSTRAINT pk_txn_mov_part PRIMARY KEY (txn_id)
USING INDEX GLOBAL;

-- Índice local sobre la fecha (alineado con la partición)
CREATE INDEX idx_txn_mov_data ON txn_movimenti_part (data_movimento)
LOCAL PARALLEL 8;

-- Índice local compuesto para consultas operativas
CREATE INDEX idx_txn_mov_cli_data
ON txn_movimenti_part (cod_cliente, data_movimento)
LOCAL PARALLEL 8;
```

La clave primaria permanece global porque las consultas por `txn_id` nunca incluyen la fecha — se necesita acceso directo. Los otros índices son locales porque se alinean con los patrones de uso: consultas por fecha, consultas por cliente+fecha.

### Paso 3: el cambio

``` sql
-- Renombrar la tabla original (respaldo)
ALTER TABLE txn_movimenti RENAME TO txn_movimenti_old;

-- Renombrar la nueva tabla
ALTER TABLE txn_movimenti_part RENAME TO txn_movimenti;

-- Reconstruir sinónimos si existen
-- Recompilar objetos invalidados
BEGIN
  FOR obj IN (SELECT object_name, object_type
              FROM   dba_objects
              WHERE  status = 'INVALID'
              AND    owner = 'BILLING') LOOP
    BEGIN
      IF obj.object_type = 'PACKAGE BODY' THEN
        EXECUTE IMMEDIATE 'ALTER PACKAGE billing.'
          || obj.object_name || ' COMPILE BODY';
      ELSIF obj.object_type IN ('PROCEDURE','FUNCTION','VIEW') THEN
        EXECUTE IMMEDIATE 'ALTER ' || obj.object_type
          || ' billing.' || obj.object_name || ' COMPILE';
      END IF;
    EXCEPTION WHEN OTHERS THEN NULL;
    END;
  END LOOP;
END;
/
```

El downtime real fue el tiempo de los dos `ALTER TABLE RENAME`: unos pocos segundos. Todo lo demás — la copia de datos, la creación de índices — ocurrió en paralelo con el sistema activo.

### Paso 4: recopilar estadísticas

``` sql
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS(
    ownname          => 'BILLING',
    tabname          => 'TXN_MOVIMENTI',
    granularity      => 'ALL',
    estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
    degree           => 8
  );
END;
/
```

El parámetro `granularity => 'ALL'` es importante: le dice a Oracle que recopile estadísticas a nivel global, de partición y de subpartición. Sin él, el optimizador podría tomar decisiones equivocadas porque no conoce la distribución de los datos dentro de las particiones individuales.

---

## Antes y después: los números

La misma consulta de facturación, después del partitioning:

``` text
------------------------------------------------------------------------
| Id  | Operation                       | Name            | Rows  | Cost |
------------------------------------------------------------------------
|   0 | SELECT STATEMENT                |                 |  125K | 12K  |
|   1 |  HASH GROUP BY                  |                 |  125K | 12K  |
|   2 |   PARTITION RANGE SINGLE        |                 |  28M  | 11K  |
|   3 |    TABLE ACCESS FULL            | TXN_MOVIMENTI   |  28M  | 11K  |
------------------------------------------------------------------------
```

Miren la operación en el paso 2: `PARTITION RANGE SINGLE`. Oracle sabe que los datos de enero están en una sola partición y lee solo esa. El full table scan que antes aterrorizaba es ahora un full **partition** scan — sobre aproximadamente 4 GB en lugar de 380.

| Métrica | Antes | Después | Variación |
|---|---|---|---|
| Tiempo consulta mensual | 4 horas | 3 minutos | -98% |
| Consistent gets | 48M | 580K | -98.8% |
| Physical reads | 12M | 95K | -99.2% |
| Tiempo GATHER_TABLE_STATS | 14 horas | 25 min (por partición) | -97% |
| Tiempo rebuild índice | 6 horas | 12 min (por partición) | -97% |
| Tamaño backup incremental | 380 GB | ~4 GB/mes | -99% |

El coste pasó de 890K a 12K. No es una mejora porcentual — es un orden de magnitud diferente.

---

## Partition pruning: la verdadera magia

El mecanismo que hace todo esto posible se llama **partition pruning**. No es algo que se configure — Oracle lo hace automáticamente cuando el predicado de la consulta coincide con la clave de partición.

Pero hay que saber cuándo funciona y cuándo no.

**Funciona** con predicados directos sobre la columna de partición:

``` sql
-- Pruning activo: Oracle lee solo la partición de enero
WHERE data_movimento BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'

-- Pruning activo: Oracle lee solo la partición específica
WHERE data_movimento = DATE '2025-03-15'
```

**No funciona** cuando la columna está dentro de una función:

``` sql
-- Pruning DESACTIVADO: Oracle debe leer todas las particiones
WHERE TRUNC(data_movimento) = DATE '2025-01-01'

-- Pruning DESACTIVADO: función sobre la columna
WHERE TO_CHAR(data_movimento, 'YYYY-MM') = '2025-01'

-- Pruning DESACTIVADO: expresión aritmética
WHERE data_movimento + 30 > SYSDATE
```

Este es el error más común que veo después de una implementación de partitioning: los desarrolladores aplican funciones a la columna de fecha sin darse cuenta de que están desactivando el pruning. Y la tabla vuelve a leerse entera.

Dediqué medio día a revisar todas las consultas de la aplicación que tocaban `TXN_MOVIMENTI`. Encontré once con `TRUNC(data_movimento)` en el `WHERE`. Once consultas que habrían ignorado el partitioning.

---

## La gestión del ciclo de vida: drop partition

Una de las ventajas más concretas del partitioning es la gestión del ciclo de vida de los datos. Antes del partitioning, archivar datos antiguos significaba un `DELETE` de miles de millones de filas — una operación que genera montañas de redo y undo, bloquea la tabla durante horas y arriesga hacer explotar el tablespace de undo.

Con el partitioning:

``` sql
-- Archivar los datos de 2016 en un tablespace de solo lectura
ALTER TABLE txn_movimenti
MOVE PARTITION p_2016_01 TABLESPACE ts_archive;

-- O, si los datos ya no se necesitan
ALTER TABLE txn_movimenti DROP PARTITION p_2016_01;
```

Un `DROP PARTITION` sobre una partición de 4 GB tarda menos de un segundo. No genera undo. No genera redo significativo. No bloquea las otras particiones. Es una operación DDL, no DML.

Configuré un job mensual que movía las particiones de más de 5 años al tablespace de archivo y las ponía en solo lectura. El cliente recuperó 120 GB de espacio activo sin eliminar un solo dato.

---

## Lo que aprendí (y los errores a evitar)

Después de quince años de partitioning Oracle, tengo una lista de cosas que me gustaría haber sabido antes.

**La clave de partición debe coincidir con el patrón de acceso.** Parece obvio, pero he visto tablas particionadas por `cod_cliente` cuando el 95% de las consultas filtra por fecha. El partitioning solo funciona si las consultas pueden hacer pruning.

**Interval partitioning es casi siempre mejor que range estático.** Con range clásico hay que crear manualmente las particiones futuras, lo que significa un job programado o un DBA que se acuerde. Con interval Oracle las crea solo. Un problema menos.

**Los índices globales son una trampa.** Funcionan bien para las consultas, pero cualquier operación DDL sobre la partición los invalida. Y reconstruir un índice global sobre 2 mil millones de filas tarda horas. Usa índices locales donde sea posible y acepta el compromiso.

**NOLOGGING no es opcional para operaciones masivas.** Sin NOLOGGING, un CTAS de 380 GB genera la misma cantidad de redo. Tu área de archivelog se llenará, la base de datos entrará en espera, y el DBA de guardia recibirá una llamada a las 3 de la mañana.

**Prueba el pruning antes de ir a producción.** No te fíes: verifica con `EXPLAIN PLAN` que cada consulta crítica haga efectivamente pruning. Un solo `TRUNC()` en el predicado equivocado y tienes un full table scan de 380 GB.

**El partitioning no sustituye los índices.** Reduce el volumen de datos a examinar, pero dentro de la partición sigues necesitando los índices correctos. Una partición mensual de 28 millones de filas sin índice sigue siendo un problema.

---

## Cuándo necesitas partitioning

No todas las tablas necesitan partitioning. Mi regla empírica:

- Bajo 10 millones de filas: probablemente no
- Entre 10 y 100 millones: depende del patrón de acceso y del ritmo de crecimiento
- Más de 100 millones: probablemente sí
- Más de mil millones: no tienes elección

Pero el momento correcto para implementarlo es antes de que se vuelva urgente. Cuando la tabla ya tiene 2 mil millones de filas, la migración es un proyecto en sí mismo. Cuando tiene 50 millones y está creciendo, es trabajo de una tarde.

Mi mayor error con el partitioning? No haberlo propuesto seis meses antes, cuando todas las señales ya estaban ahí.
