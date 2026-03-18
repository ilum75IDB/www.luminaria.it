---
title: "IST"
description: "Incremental State Transfer — mecanismo de Galera Cluster para transferir solo las transacciones faltantes a un nodo que reingresa al cluster."
translationKey: "glossary_ist"
aka: "Incremental State Transfer"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**IST** (Incremental State Transfer) es el mecanismo por el cual un nodo Galera que reingresa al cluster después de una ausencia breve recibe solo las transacciones faltantes, sin necesidad de descargar el dataset entero.

## Cómo funciona

Cuando un nodo se reconecta al cluster, el donante verifica si las transacciones faltantes aún están disponibles en su gcache (Galera cache). Si el gap está cubierto por el gcache, se ejecuta un IST: solo las transacciones faltantes se envían al nodo, que las aplica y vuelve al estado Synced. Si el gap supera el gcache, Galera recurre a un SST completo.

## Para qué sirve

IST hace que el reingreso de un nodo al cluster sea mucho más rápido que un SST completo. Un nodo que ha estado offline unos minutos o unas horas puede volver a estar operativo en segundos, sin impacto en el rendimiento del cluster.

## Cuándo se usa

IST se activa automáticamente cuando las condiciones lo permiten. El tamaño del gcache (`gcache.size`) determina cuántas transacciones el cluster puede mantener en memoria para soportar IST. Un gcache más grande permite tiempos de inactividad más largos de un nodo sin necesidad de SST.
