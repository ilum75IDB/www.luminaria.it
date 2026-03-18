---
title: "Huge Pages"
description: "Pagini de memorie de 2 MB (în loc de cele standard de 4 KB) care reduc drastic presiunea pe MMU și TLB, îmbunătățind performanțele Oracle pe Linux."
translationKey: "glossary_huge-pages"
aka: "HugePages / Large Pages"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

**Huge Pages** sunt pagini de memorie de 2 MB, față de cele standard de 4 KB ale Linux. Pentru o SGA Oracle de 64 GB, trecerea de la pagini de 4 KB (16,7 milioane de pagini) la Huge Pages de 2 MB (32.768 pagini) reduce de 500 de ori numărul de intrări în Page Table.

## Cum funcționează

Se configurează prin parametrul kernel `vm.nr_hugepages` în `/etc/sysctl.d/`. Numărul necesar se calculează împărțind dimensiunea SGA la 2 MB și adăugând o marjă de 1,5%. După repornirea instanței Oracle, SGA este alocată în Huge Pages, verificabil din `/proc/meminfo`.

## La ce servește

Reduc presiunea pe TLB-ul (Translation Lookaside Buffer) CPU-ului, care poate memora doar câteva mii de traduceri de adrese. Cu pagini normale, TLB-ul face overflow constant iar MMU-ul trebuie să gestioneze milioane de traduceri — cu impact măsurabil pe latch free waits și library cache contention.

## De ce contează

Este singurul parametru cu cel mai mare impact pentru Oracle pe Linux, și cel ignorat cel mai des. Asistentul de instalare nu îl configurează, documentația e într-o notă MOS, iar sistemul "funcționează și fără." Dar metricile înainte/după vorbesc clar: library cache hit ratio de la 92% la 99,7%, CPU de la 78% la 41%.
