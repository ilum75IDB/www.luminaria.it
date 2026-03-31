---
title: "Binary log"
description: "Registro binario secuencial de MySQL que rastrea todas las modificaciones de datos, usado para la replicación y el point-in-time recovery."
translationKey: "glossary_binary-log"
aka: "binlog"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/binary-log-mysql"
---

El **binary log** (o binlog) es un registro secuencial en formato binario donde MySQL escribe todos los eventos que modifican datos: INSERT, UPDATE, DELETE y operaciones DDL. Los archivos se numeran progresivamente (`mysql-bin.000001`, `mysql-bin.000002`, etc.) y se gestionan mediante un archivo índice.

## Cómo funciona

Desde MySQL 8.0 el binary log está habilitado por defecto mediante el parámetro `log_bin`. MySQL crea un nuevo archivo binlog cuando el servidor se inicia, cuando el archivo actual alcanza `max_binlog_size`, o cuando se ejecuta `FLUSH BINARY LOGS`. Soporta tres formatos de registro: STATEMENT (registra las instrucciones SQL), ROW (registra los cambios fila por fila) y MIXED (elección automática).

## Para qué sirve

El binary log tiene dos funciones fundamentales:

- **Replicación**: en una arquitectura master-slave, el slave lee los binlog del master para replicar las mismas operaciones
- **Point-in-time recovery**: después de restaurar un backup, los binlog permiten reaplicar los cambios hasta un momento preciso

## Cuándo se usa

El binary log está activo por defecto en cualquier instalación MySQL 8.0+. La gestión activa (retención, purge, monitorización del espacio) es necesaria para evitar que los archivos acumulados llenen el disco. El comando `PURGE BINARY LOGS` es la forma correcta de eliminar archivos obsoletos.
