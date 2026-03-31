---
title: "GTID"
description: "Global Transaction Identifier — identificador único asignado a cada transacción en MySQL para simplificar la gestión de la replicación."
translationKey: "glossary_gtid"
aka: "Global Transaction Identifier"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/mysqldump-mysqlpump-mydumper"
---

**GTID** (Global Transaction Identifier) es un identificador único asignado automáticamente a cada transacción confirmada en un servidor MySQL. El formato es `server_uuid:transaction_id` — por ejemplo `3E11FA47-71CA-11E1-9E33-C80AA9429562:23`.

## Cómo funciona

Cuando el GTID está habilitado (`gtid_mode = ON`), cada transacción recibe un identificador que la hace rastreable en cualquier servidor del cluster de replicación. La réplica sabe exactamente qué transacciones ya ejecutó y cuáles necesita recibir, sin necesidad de especificar manualmente posiciones de binlog (archivo + offset).

El conjunto de todos los GTID ejecutados en un servidor se almacena en la variable `gtid_executed`. Cuando una réplica se conecta al origen, compara su propio `gtid_executed` con el del origen para determinar qué transacciones faltan.

## Para qué sirve

El GTID simplifica radicalmente la gestión de la replicación MySQL:

- **Failover automático**: cuando el origen cae, una réplica puede convertirse en el nuevo origen y las demás réplicas se realinean automáticamente
- **Verificación de consistencia**: es posible verificar si dos servidores han ejecutado exactamente las mismas transacciones
- **Backup y restore**: herramientas como mysqldump y mydumper deben gestionar correctamente los GTID para evitar conflictos de replicación después del restore

## Cuándo crea problemas

Los GTID requieren atención durante las operaciones de backup y restore. Si se restaura un dump en un servidor con GTID activo sin configurar correctamente `--set-gtid-purged`, se pueden generar conflictos que rompen la cadena de replicación.
