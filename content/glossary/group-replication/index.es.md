---
title: "Group Replication"
description: "Mecanismo nativo de MySQL para la replicación síncrona multi-nodo con failover automático y gestión de quórum."
translationKey: "glossary_group_replication"
aka: "MySQL Group Replication / InnoDB Cluster"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/binary-log-mysql"
---

**Group Replication** es el mecanismo nativo de MySQL para crear clusters de alta disponibilidad con replicación síncrona entre múltiples nodos. A diferencia de la replicación clásica (asíncrona, master-slave), Group Replication garantiza que cada transacción sea confirmada por la mayoría de los nodos antes de considerarse committed.

## Cómo funciona

Los nodos se comunican mediante un protocolo de grupo (GCS — Group Communication System) que gestiona el consenso distribuido. Cada nodo mantiene una copia completa de los datos. Las transacciones son certificadas por el grupo: si no hay conflictos, se aplican en todos los nodos. Si hay un conflicto, la transacción se revierte en el nodo que la originó.

## Modos operativos

MySQL Group Replication soporta dos modos: **single-primary** (solo un nodo acepta escrituras, los demás son de solo lectura) y **multi-primary** (todos los nodos aceptan escrituras). El modo single-primary es el más usado en producción porque evita los conflictos de escritura concurrente.

## Por qué es crítico

Group Replication gestiona automáticamente el failover: si el primary cae, el cluster elige un nuevo primary entre los secondary en pocos segundos. Esto lo hace adecuado para entornos que requieren alta disponibilidad sin intervención manual. Requiere un mínimo de tres nodos para mantener el quórum.
