---
title: "I/O Scheduler"
description: "Componentă a kernelului Linux care decide ordinea în care cererile de I/O sunt trimise către disc, cu impact direct asupra performanțelor bazei de date."
translationKey: "glossary_io-scheduler"
aka: "Scheduler de I/O Linux"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

**I/O Scheduler** este componenta kernelului Linux care gestionează coada cererilor de citire și scriere către dispozitivele bloc (discuri). Decide ordinea de execuție a cererilor pentru a optimiza throughput-ul și a minimiza latența.

## Cum funcționează

Linux oferă mai mulți scheduleri: `cfq` (Completely Fair Queuing, pentru desktop), `deadline`/`mq-deadline` (pentru servere și baze de date), `noop`/`none` (pentru SSD/NVMe). Pentru Oracle recomandarea este `deadline`, care servește cererile minimizând seek-urile discului. Se configurează prin `/sys/block/sdX/queue/scheduler` și se face permanent prin GRUB.

## La ce servește

`cfq`-ul implicit distribuie I/O-ul egal între procese — ideal pentru desktop, dezastruos pentru o bază de date care are nevoie de prioritate pe cererile I/O critice. `deadline` garantează că nicio cerere nu rămâne în coadă prea mult timp, reducând latența `db file sequential read`.

## Ce poate merge prost

Lăsarea valorii implicite (`cfq` sau `bfq` pe unele sisteme) înseamnă că Oracle concurează pentru I/O cu toate celelalte procese ale sistemului. Pe un server dedicat bazei de date este o risipă: baza de date ar trebui să aibă prioritate absolută pe operațiunile de disc.
