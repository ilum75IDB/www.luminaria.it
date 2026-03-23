---
title: "RAC"
description: "Real Application Clusters — tecnologia Oracle que permite a multiples instancias acceder simultaneamente a la misma base de datos, garantizando alta disponibilidad y escalabilidad."
translationKey: "glossary_rac"
aka: "Real Application Clusters"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**RAC** (Real Application Clusters) es la tecnologia Oracle que permite a multiples instancias de base de datos acceder simultaneamente al mismo almacenamiento compartido. Si un nodo falla, los demas continuan atendiendo solicitudes sin interrupcion — el failover es transparente para las aplicaciones.

## Como funciona

Un cluster RAC esta compuesto por dos o mas servidores (nodos) conectados entre si mediante una red privada de alta velocidad (interconnect) y almacenamiento compartido (tipicamente ASM — Automatic Storage Management). Cada nodo ejecuta su propia instancia Oracle, pero todos acceden a los mismos datafiles.

El protocolo **Cache Fusion** gestiona la coherencia de datos entre nodos: cuando un bloque modificado por un nodo es necesario en otro, se transfiere directamente por el interconnect sin pasar por disco.

## Alta disponibilidad

Si un nodo cae, las sesiones activas se transfieren automaticamente a los nodos restantes mediante **TAF** (Transparent Application Failover) o **Application Continuity**. El tiempo de failover depende de la configuracion pero tipicamente es del orden de segundos.

## Implicaciones de licensing

RAC es una opcion de la Enterprise Edition con un coste de licensing significativo. En una migracion cloud, el conteo de licencias RAC es uno de los aspectos mas delicados: en OCI con BYOL las licencias on-premises se reutilizan, en otros proveedores cloud el coste puede multiplicarse.

## Cuando es realmente necesario

RAC esta justificado cuando se necesita alta disponibilidad con failover automatico y escalabilidad horizontal. Para entornos con pocos usuarios o requisitos de uptime estandar, un nodo unico con Data Guard es a menudo una solucion mas simple y economica.
