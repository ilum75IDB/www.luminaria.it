---
title: "Oracle pe Linux: parametrii kernel pe care nimeni nu-i configurează"
description: "Un client cu Oracle 19c pe Linux și performanță dezamăgitoare. Instalare implicită, fără tuning. Huge Pages, semafoare, I/O scheduler, THP și limite de securitate: tot ce lipsea — cu cifrele de dinainte și după."
date: "2026-02-24T10:00:00+01:00"
draft: false
translationKey: "oracle_linux_kernel"
tags: ["oracle", "linux", "kernel", "tuning", "hugepages", "performance"]
categories: ["oracle"]
image: "oracle-linux-kernel.cover.jpg"
---

Clientul era o companie de logistică cu Oracle 19c Enterprise Edition pe Oracle Linux 8. Șaizeci de utilizatori concurenți, o aplicație ERP personalizată, circa 400 GB de date. Serverul era un Dell PowerEdge cu 128 GB de RAM și 32 de core-uri.

Plângerile erau vagi dar persistente: "Sistemul e lent." "Query-urile de dimineață durează dublu față de acum două luni." "Din când în când se blochează totul câteva secunde."

Când m-am conectat pe server, primul lucru pe care l-am verificat nu a fost baza de date. A fost sistemul de operare.

``` bash
cat /proc/meminfo | grep -i huge
sysctl vm.nr_hugepages
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Rezultat: zero Huge Pages configurate, Transparent Huge Pages active, parametrii kernel la valorile implicite. Instalarea Oracle fusese făcută cu asistentul, sistemul de operare nu fusese atins.

Asta era problema. Nu era Oracle. Era Linux care nu fusese pregătit pentru Oracle.

------------------------------------------------------------------------

## 🔍 Diagnosticul

Înainte de a schimba orice, am măsurat starea curentă. Ai nevoie de cifre, nu de impresii.

``` bash
# Starea SGA
sqlplus -s / as sysdba <<SQL
SELECT name, value/1024/1024 AS mb
FROM   v$sgainfo
WHERE  name IN ('Maximum SGA Size', 'Free SGA Memory Available');
SQL

# Utilizarea memoriei sistemului
free -h

# Parametrii kernel curenți
sysctl -a | grep -E "kernel.sem|kernel.shm|vm.nr_hugepages|vm.swappiness"

# I/O scheduler-ul în uz
cat /sys/block/sda/queue/scheduler

# Limitele utilizatorului oracle
su - oracle -c "ulimit -a"
```

Iată ce am găsit:

| Parametru | Valoare curentă | Valoare recomandată |
|---|---|---|
| SGA Target | 64 GB | 64 GB (ok) |
| vm.nr_hugepages | 0 | 33280 |
| Transparent Huge Pages | always | never |
| vm.swappiness | 60 | 1 |
| kernel.shmmax | 33554432 (32 MB) | 68719476736 (64 GB) |
| kernel.shmall | 2097152 | 16777216 |
| kernel.sem | 250 32000 100 128 | 250 32000 100 256 |
| I/O scheduler | mq-deadline | deadline (ok) |
| oracle nofile | 1024 | 65536 |
| oracle nproc | 4096 | 16384 |
| oracle memlock | 65536 KB | unlimited |

Aproape totul era greșit. Nu din eroare — din omisiune. Nimeni nu se obosise să configureze sistemul de operare după instalare.

------------------------------------------------------------------------

## 📦 Huge Pages: parametrul care schimbă totul

Huge Pages sunt parametrul individual cu cel mai mare impact pentru Oracle pe Linux. Și sunt totodată cel mai des ignorat.

### De ce contează

Implicit, Linux gestionează memoria în pagini de 4 KB. O SGA de 64 GB înseamnă aproximativ 16,7 milioane de pagini. Fiecare pagină are o intrare în Page Table, iar sistemul trebuie să traducă adrese virtuale în fizice pentru fiecare. TLB-ul (Translation Lookaside Buffer) al CPU-ului poate memora doar câteva mii de traduceri — restul e gestionat de MMU, care e lentă.

Huge Pages sunt pagini de 2 MB. Aceeași SGA de 64 GB devine 32.768 de pagini. TLB-ul face față, presiunea pe MMU scade, performanța se îmbunătățește.

### Cum se configurează

Am calculat numărul de Huge Pages necesare:

``` bash
# SGA = 64 GB = 65536 MB
# Fiecare Huge Page = 2 MB
# Pagini necesare = 65536 / 2 = 32768
# Adaug 1,5% marjă → 33280

echo "vm.nr_hugepages = 33280" >> /etc/sysctl.d/99-oracle.conf
sysctl -p /etc/sysctl.d/99-oracle.conf
```

Verificare:

``` bash
grep -i huge /proc/meminfo
```

Output așteptat:

```
HugePages_Total:   33280
HugePages_Free:    33280
HugePages_Rsvd:        0
Hugepagesize:       2048 kB
```

După repornirea instanței Oracle, SGA-ul se alocă în Huge Pages:

```
HugePages_Total:   33280
HugePages_Free:      512
HugePages_Rsvd:      480
```

Diferența e măsurabilă: `latch free` waits și `library cache` contention scad dramatic.

------------------------------------------------------------------------

## 🧱 Memorie partajată și semafoare

Oracle folosește memoria partajată a kernel-ului pentru SGA. Dacă limitele sunt prea mici, instanța nu poate aloca memoria cerută — sau mai rău, fragmentează alocarea.

``` bash
cat >> /etc/sysctl.d/99-oracle.conf << 'SYSCTL'
# Shared memory
kernel.shmmax = 68719476736
kernel.shmall = 16777216
kernel.shmmni = 4096

# Semaphores: SEMMSL SEMMNS SEMOPM SEMMNI
kernel.sem = 250 32000 100 256
SYSCTL

sysctl -p /etc/sysctl.d/99-oracle.conf
```

| Parametru | Semnificație | Valoare |
|---|---|---|
| shmmax | Dimensiunea maximă a unui segment de memorie partajată | 64 GB |
| shmall | Pagini totale de memorie partajată alocabile | 64 GB în pagini de 4K |
| shmmni | Numărul maxim de segmente de memorie partajată | 4096 |
| sem | SEMMSL, SEMMNS, SEMOPM, SEMMNI | 250 32000 100 256 |

Nu sunt valori magice. Sunt dimensionate pentru SGA-ul bazei de date. Dacă SGA-ul se schimbă, parametrii trebuie recalculați.

------------------------------------------------------------------------

## 💾 I/O Scheduler

Default-ul pe RHEL/Oracle Linux 8 cu dispozitive NVMe este `none` sau `mq-deadline`. Pentru discuri SAS/SATA tradiționale, default-ul poate fi `bfq` sau `cfq`.

Pentru Oracle, recomandarea este `deadline` (sau `mq-deadline` pe kernel-uri mai noi):

``` bash
# Verificare setare curentă
cat /sys/block/sda/queue/scheduler

# Dacă nu e deadline/mq-deadline, setează-l
echo "deadline" > /sys/block/sda/queue/scheduler

# Fă-l permanent prin GRUB
grubby --update-kernel=ALL --args="elevator=deadline"
```

`cfq` (Completely Fair Queuing) e conceput pentru workload-uri desktop — distribuie I/O-ul echitabil între procese. Dar Oracle nu are nevoie de echitate: are nevoie ca cererile I/O să fie servite în ordinea care minimizează seek-urile. `deadline` face exact asta.

------------------------------------------------------------------------

## 🚫 Dezactivarea Transparent Huge Pages

Acesta e parametrul cel mai insidios. Transparent Huge Pages (THP) e o funcție a kernel-ului care **pare** o idee bună: kernel-ul promovează automat paginile normale la pagini mari.

Pentru Oracle e un dezastru. Procesul `khugepaged` lucrează în fundal pentru a compacta paginile, cauzând vârfuri de latență imprevizibile — acele "blocaje de câteva secunde" pe care clientul le raporta.

Oracle spune explicit în documentație: **dezactivați THP**.

``` bash
# Verificare stare curentă
cat /sys/kernel/mm/transparent_hugepage/enabled
# Output tipic: [always] madvise never

# Dezactivare la runtime
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Permanentizare prin GRUB
grubby --update-kernel=ALL --args="transparent_hugepage=never"
```

După reboot, verificare:

``` bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# Output așteptat: always madvise [never]
```

Diferența e clară: micro-blocajele aleatorii dispar.

------------------------------------------------------------------------

## 🔒 Limite de securitate

Utilizatorul `oracle` are nevoie de limite ridicate pentru descriptori de fișiere deschise, procese și memorie blocabilă. Default-urile Linux sunt gândite pentru utilizatori interactivi, nu pentru software care gestionează sute de conexiuni simultane.

``` bash
cat >> /etc/security/limits.d/99-oracle.conf << 'LIMITS'
oracle   soft   nofile    65536
oracle   hard   nofile    65536
oracle   soft   nproc     16384
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   soft   memlock   unlimited
oracle   hard   memlock   unlimited
LIMITS
```

| Limită | Default | Recomandat | De ce |
|---|---|---|---|
| nofile | 1024 | 65536 | Oracle deschide un descriptor pentru fiecare datafile, redo log, archive log |
| nproc | 4096 | 16384 | Fiecare proces Oracle e un proces OS separat |
| memlock | 65536 KB | unlimited | Necesar pentru blocarea SGA-ului în Huge Pages |
| stack | 8192 KB | 10240-32768 KB | PL/SQL recursiv profund poate epuiza stack-ul |

Setarea `memlock unlimited` e critică: fără ea, Oracle nu poate bloca SGA-ul în Huge Pages, făcând inutilă configurarea anterioară.

------------------------------------------------------------------------

## ⚡ Swappiness

Valoarea implicită a `vm.swappiness` este 60. Asta înseamnă că Linux începe să facă swap când presiunea pe memorie e încă scăzută. Pentru un server de baze de date dedicat, asta e inacceptabil: vrei ca SGA-ul să rămână în RAM, mereu.

``` bash
echo "vm.swappiness = 1" >> /etc/sysctl.d/99-oracle.conf
sysctl -p /etc/sysctl.d/99-oracle.conf
```

Nu zero — unu. Valoarea zero dezactivează complet swap-ul, ceea ce poate declanșa OOM killer în situații de presiune extremă. Valoarea unu îi spune kernel-ului: "Fă swap doar când chiar nu mai e alternativă."

------------------------------------------------------------------------

## 📊 Înainte și după

După aplicarea tuturor configurărilor și repornirea instanței Oracle, am refăcut măsurătorile.

| Metrică | Înainte | După | Variație |
|---|---|---|---|
| SGA în Huge Pages | Nu | Da | — |
| Library cache hit ratio | 92,3% | 99,7% | +7,4% |
| Buffer cache hit ratio | 94,1% | 99,2% | +5,1% |
| Timp mediu așteptare (db file sequential read) | 8,2 ms | 1,4 ms | -83% |
| Micro-blocaje aleatorii (>1s) | 5-8 pe zi | 0 | -100% |
| Timp mediu batch matinal | 47 min | 22 min | -53% |
| Utilizare CPU medie | 78% | 41% | -47% |
| Swap utilizat | 3,2 GB | 0 MB | -100% |

Cifrele vorbesc de la sine. Aceeași mașină, aceeași bază de date, aceeași sarcină de lucru. Singura diferență: sistemul de operare a fost configurat să-și facă treaba.

------------------------------------------------------------------------

## 📋 Checklist final

Pentru cine dorește un rezumat operațional, iată checklist-ul complet:

``` bash
# /etc/sysctl.d/99-oracle.conf
vm.nr_hugepages = 33280
vm.swappiness = 1
kernel.shmmax = 68719476736
kernel.shmall = 16777216
kernel.shmmni = 4096
kernel.sem = 250 32000 100 256
```

``` bash
# /etc/security/limits.d/99-oracle.conf
oracle   soft   nofile    65536
oracle   hard   nofile    65536
oracle   soft   nproc     16384
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   soft   memlock   unlimited
oracle   hard   memlock   unlimited
```

``` bash
# GRUB
grubby --update-kernel=ALL --args="transparent_hugepage=never elevator=deadline"
```

Zece minute de configurare. Fără cost hardware. Fără licențe suplimentare.

Dar nimeni nu face asta, pentru că asistentul nu întreabă, documentația e îngropată într-o notă MOS, iar sistemul "funcționează și fără." Funcționează. Prost. Iar vina cade mereu pe Oracle, niciodată pe faptul că nimeni nu a pregătit terenul.

O bază de date e la fel de bună ca sistemul de operare pe care rulează. Iar un sistem de operare lăsat la valorile implicite e un sistem de operare care lucrează împotriva ta.

------------------------------------------------------------------------

## Glosar

**[Huge Pages](/ro/glossary/huge-pages/)** — Pagini de memorie de 2 MB (în loc de cele standard de 4 KB) care reduc drastic presiunea pe MMU și TLB, îmbunătățind performanțele Oracle pe Linux.

**[THP](/ro/glossary/thp/)** — Transparent Huge Pages — funcție a kernelului Linux care promovează automat paginile normale la pagini mari, dar care cauzează latențe imprevizibile și trebuie dezactivată pentru Oracle.

**[SGA](/ro/glossary/sga/)** — System Global Area — zona de memorie partajată a Oracle Database care conține buffer cache, shared pool, redo log buffer și alte structuri critice pentru performanță.

**[I/O Scheduler](/ro/glossary/io-scheduler/)** — Componentă a kernelului Linux care decide ordinea în care cererile de I/O sunt trimise către disc, cu impact direct asupra performanțelor bazei de date.

**[Swappiness](/ro/glossary/swappiness/)** — Parametru kernel Linux (vm.swappiness) care controlează propensiunea sistemului de a muta pagini de memorie în swap, critic pentru serverele de baze de date.
