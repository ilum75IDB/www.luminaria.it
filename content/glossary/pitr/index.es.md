---
title: "PITR"
description: "Point-in-Time Recovery — técnica de restauración que permite llevar una base de datos a un momento preciso en el tiempo, combinando backups y logs de transacciones."
translationKey: "glossary_pitr"
aka: "Point-in-Time Recovery"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**PITR** (Point-in-Time Recovery) es una técnica de restauración que permite llevar una base de datos a cualquier momento en el tiempo, no solo al momento del backup. Se basa en la combinación de un backup completo y los logs de transacciones (binary log en MySQL, WAL en PostgreSQL, redo log en Oracle).

## Cómo funciona

El proceso se articula en dos fases:

1. **Restore del backup**: se restaura la base de datos al último backup disponible
2. **Replay de los logs**: se reaplican los logs de transacciones desde el momento del backup hasta el momento deseado, excluyendo el evento que causó el problema

En MySQL, la herramienta `mysqlbinlog` extrae los eventos de los binary log y los reproduce sobre la base de datos restaurada.

## Para qué sirve

PITR es esencial cuando se produce un error humano (DROP TABLE, DELETE sin WHERE, UPDATE masivo equivocado) y se necesita restaurar la base de datos al estado inmediatamente anterior al error, sin perder las horas de trabajo entre el último backup y el incidente.

## Cuándo se usa

PITR requiere que el binary log esté activo y que los archivos binlog no hayan sido eliminados. La retención de los binlog debe cubrir al menos el doble del intervalo entre dos backups consecutivos para garantizar una cobertura PITR completa.
