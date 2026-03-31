---
title: "Single-primary"
description: "Modo de MySQL Group Replication en el que solo un nodo acepta escrituras mientras los demás son de solo lectura con failover automático."
translationKey: "glossary_single_primary"
aka: "Single-primary mode"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
---

**Single-primary** es el modo operativo más común de MySQL Group Replication, donde solo un nodo del cluster — el primary — acepta operaciones de escritura. Los demás nodos (secondary) son de solo lectura (`read_only=ON`, `super_read_only=ON`) y reciben las modificaciones a través de la replicación síncrona del grupo.

## Cómo funciona

El parámetro `group_replication_single_primary_mode=ON` activa este modo. El primary es el único nodo con `read_only=OFF`. Si el primary se detiene o se vuelve inaccesible, el cluster ejecuta una elección automática y uno de los secondary se convierte en el nuevo primary en pocos segundos.

## Por qué se usa

El modo single-primary evita los conflictos de escritura concurrente típicos del multi-primary. En producción la mayoría de los clusters MySQL usan este modo porque es más predecible: las aplicaciones escriben en un solo endpoint, la replicación es lineal y el debugging es más sencillo.

## Qué puede salir mal

Cuando se detiene el primary por mantenimiento, el cluster realiza un failover automático. Durante esos segundos las conexiones activas pueden caerse y las transacciones en curso pueden fallar. Es una interrupción breve pero debe comunicarse. La regla práctica: en una intervención de mantenimiento sobre un cluster single-primary, los secondary se tocan primero, el primary de último.
