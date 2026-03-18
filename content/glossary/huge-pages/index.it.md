---
title: "Huge Pages"
description: "Pagine di memoria da 2 MB (invece dei 4 KB standard) che riducono drasticamente la pressione sulla MMU e sul TLB, migliorando le performance di Oracle su Linux."
translationKey: "glossary_huge-pages"
aka: "HugePages / Large Pages"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

Le **Huge Pages** sono pagine di memoria da 2 MB, contro i 4 KB standard di Linux. Per un'SGA Oracle da 64 GB, passare da pagine da 4 KB (16,7 milioni di pagine) a Huge Pages da 2 MB (32.768 pagine) riduce di 500 volte il numero di entry nella Page Table.

## Come funziona

Si configurano tramite il parametro kernel `vm.nr_hugepages` in `/etc/sysctl.d/`. Il numero necessario si calcola dividendo la dimensione della SGA per 2 MB e aggiungendo un margine dell'1,5%. Dopo il riavvio dell'istanza Oracle, la SGA viene allocata nelle Huge Pages, verificabile da `/proc/meminfo`.

## A cosa serve

Riducono la pressione sul TLB (Translation Lookaside Buffer) della CPU, che può memorizzare solo poche migliaia di traduzioni indirizzo. Con pagine normali, il TLB va in overflow costante e la MMU deve gestire milioni di traduzioni — con un impatto misurabile su latch free wait e library cache contention.

## Perché è critico

È il singolo parametro più impattante per Oracle su Linux, e quello ignorato più spesso. L'installazione wizard non lo configura, la documentazione è in una nota MOS, e il sistema "funziona anche senza." Ma le metriche prima/dopo parlano chiaro: library cache hit ratio dal 92% al 99,7%, CPU dal 78% al 41%.
