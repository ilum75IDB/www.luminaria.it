---
title: "I/O Scheduler"
description: "Componente del kernel Linux que decide el orden en que las solicitudes de I/O se envían al disco, con impacto directo en el rendimiento de la base de datos."
translationKey: "glossary_io-scheduler"
aka: "Scheduler de I/O Linux"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

El **I/O Scheduler** es el componente del kernel Linux que gestiona la cola de solicitudes de lectura y escritura hacia los dispositivos de bloque (discos). Decide el orden de ejecución de las solicitudes para optimizar el throughput y minimizar la latencia.

## Cómo funciona

Linux ofrece varios schedulers: `cfq` (Completely Fair Queuing, para escritorio), `deadline`/`mq-deadline` (para servidores y bases de datos), `noop`/`none` (para SSD/NVMe). Para Oracle la recomendación es `deadline`, que sirve las solicitudes minimizando los seeks del disco. Se configura vía `/sys/block/sdX/queue/scheduler` y se hace permanente vía GRUB.

## Para qué sirve

El `cfq` por defecto distribuye el I/O equitativamente entre los procesos — ideal para un escritorio, pésimo para una base de datos que necesita prioridad en las solicitudes I/O críticas. `deadline` garantiza que ninguna solicitud permanezca en cola demasiado tiempo, reduciendo la latencia de los `db file sequential read`.

## Qué puede salir mal

Dejar el valor por defecto (`cfq` o `bfq` en algunos sistemas) significa que Oracle compite por el I/O con todos los demás procesos del sistema. En un servidor dedicado a la base de datos es un desperdicio: la base de datos debería tener prioridad absoluta en las operaciones de disco.
