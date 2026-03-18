---
title: "CDC"
description: "Change Data Capture — técnica para interceptar y propagar los cambios en los datos en tiempo real, frecuentemente basada en la lectura de los logs de transacciones."
translationKey: "glossary_cdc"
aka: "Change Data Capture"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**CDC** (Change Data Capture) es una técnica para interceptar las modificaciones de datos (INSERT, UPDATE, DELETE) en el momento en que ocurren y propagarlas hacia otros sistemas en tiempo real o casi real. A diferencia de los enfoques batch tradicionales (ETL periódicos), el CDC captura los cambios de forma continua e incremental.

## Cómo funciona

El enfoque más difundido es el **log-based CDC**: un componente externo lee los logs de transacciones de la base de datos (binary log en MySQL, WAL en PostgreSQL, redo log en Oracle) y convierte los eventos en un flujo de datos consumible por otros sistemas. Herramientas como Debezium, Maxwell y Canal implementan este enfoque para MySQL leyendo directamente los binary log.

## Para qué sirve

El CDC se usa para:

- Sincronizar datos entre bases de datos diferentes en tiempo real
- Alimentar data warehouses y data lakes con actualizaciones incrementales
- Poblar cachés e índices de búsqueda (Elasticsearch, Redis)
- Implementar arquitecturas event-driven y microservicios

## Cuándo se usa

El CDC requiere que el binary log esté activo y en formato ROW (que registra los cambios fila por fila). Desactivar los binary log o usar el formato STATEMENT elimina la posibilidad de utilizar herramientas de CDC, haciendo imposible la integración en tiempo real con sistemas externos.
