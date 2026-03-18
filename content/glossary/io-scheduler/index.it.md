---
title: "I/O Scheduler"
description: "Componente del kernel Linux che decide l'ordine in cui le richieste di I/O vengono inviate al disco, con impatto diretto sulle performance del database."
translationKey: "glossary_io-scheduler"
aka: "Scheduler di I/O Linux"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

L'**I/O Scheduler** è il componente del kernel Linux che gestisce la coda delle richieste di lettura e scrittura verso i dispositivi a blocchi (dischi). Decide l'ordine di esecuzione delle richieste per ottimizzare il throughput e minimizzare la latenza.

## Come funziona

Linux offre diversi scheduler: `cfq` (Completely Fair Queuing, per desktop), `deadline`/`mq-deadline` (per server e database), `noop`/`none` (per SSD/NVMe). Per Oracle la raccomandazione è `deadline`, che serve le richieste minimizzando i seek del disco. Si configura via `/sys/block/sdX/queue/scheduler` e si rende permanente via GRUB.

## A cosa serve

Il `cfq` di default distribuisce equamente l'I/O tra i processi — ideale per un desktop, pessimo per un database che ha bisogno di priorità sulle richieste I/O critiche. `deadline` garantisce che nessuna richiesta resti in coda troppo a lungo, riducendo la latenza dei `db file sequential read`.

## Cosa può andare storto

Lasciare il default (`cfq` o `bfq` su alcuni sistemi) significa che Oracle compete per l'I/O con tutti gli altri processi del sistema. Su un server dedicato al database è uno spreco: il database dovrebbe avere priorità assoluta sulle operazioni disco.
