---
title: "SGA"
description: "System Global Area — area di memoria condivisa di Oracle Database che contiene buffer cache, shared pool, redo log buffer e altre strutture critiche per le performance."
translationKey: "glossary_sga"
aka: "System Global Area"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

La **SGA** (System Global Area) è l'area di memoria condivisa principale di Oracle Database. Contiene le strutture dati fondamentali: buffer cache (pagine dati lette da disco), shared pool (piani di esecuzione e dizionario dati), redo log buffer e large pool.

## Come funziona

La dimensione della SGA è controllata dal parametro `SGA_TARGET` o `SGA_MAX_SIZE`. Oracle alloca la SGA all'avvio dell'istanza in memoria condivisa del sistema operativo. I parametri kernel Linux `shmmax` e `shmall` devono essere dimensionati per permettere l'allocazione completa della SGA.

## A cosa serve

Tutta l'attività di lettura e scrittura del database passa attraverso la SGA. Un buffer cache efficiente evita letture fisiche da disco. Uno shared pool ben dimensionato evita re-parsing delle query. La SGA è il cuore delle performance Oracle — e deve risiedere in Huge Pages per massimizzare l'efficienza.

## Perché è critico

Una SGA non allocata in Huge Pages significa milioni di entry nella Page Table e overflow costante del TLB. Il risultato sono latch free wait, library cache contention e CPU elevata. Configurare le Huge Pages e il parametro `memlock unlimited` per l'utente oracle è il prerequisito per qualsiasi tuning serio.
