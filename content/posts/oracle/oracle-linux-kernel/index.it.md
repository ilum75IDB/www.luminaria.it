---
title: "Oracle su Linux: i parametri kernel che nessuno configura"
description: "Un cliente con Oracle 19c su Linux e performance deludenti. Installazione di default, nessun tuning. Huge Pages, semafori, I/O scheduler, THP e limiti di sicurezza: tutto quello che mancava — con i numeri prima e dopo."
date: "2026-02-24T10:00:00+01:00"
draft: false
translationKey: "oracle_linux_kernel"
tags: ["oracle", "linux", "kernel", "tuning", "hugepages", "performance"]
categories: ["oracle"]
image: "oracle-linux-kernel.cover.jpg"
---

Il cliente era una società di logistica con un Oracle 19c Enterprise Edition su Oracle Linux 8. Sessanta utenti concorrenti, un gestionale custom, circa 400 GB di dati. Il server era un Dell PowerEdge con 128 GB di RAM e 32 core.

Le lamentele erano vaghe ma persistenti: "Il sistema è lento." "Le query del mattino ci mettono il doppio rispetto a due mesi fa." "Ogni tanto si blocca tutto per qualche secondo."

Quando sono entrato sul server, la prima cosa che ho verificato non è stata il database. È stato il sistema operativo.

``` bash
cat /proc/meminfo | grep -i huge
sysctl vm.nr_hugepages
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Risultato: zero Huge Pages configurate, Transparent Huge Pages attive, parametri kernel tutti ai valori di default. L'installazione di Oracle era stata fatta col wizard, il sistema operativo non era stato toccato.

Ecco il problema. Non era Oracle. Era Linux che non era stato preparato per Oracle.

------------------------------------------------------------------------

## 🔍 La diagnosi

Prima di cambiare qualsiasi cosa, ho misurato lo stato attuale. Servono numeri, non impressioni.

``` bash
# Stato della SGA
sqlplus -s / as sysdba <<SQL
SELECT name, value/1024/1024 AS mb
FROM   v$sgainfo
WHERE  name IN ('Maximum SGA Size', 'Free SGA Memory Available');
SQL

# Utilizzo memoria del sistema
free -h

# Parametri kernel correnti
sysctl -a | grep -E "kernel.sem|kernel.shm|vm.nr_hugepages|vm.swappiness"

# I/O scheduler in uso
cat /sys/block/sda/queue/scheduler

# Limiti utente oracle
su - oracle -c "ulimit -a"
```

Ecco cosa ho trovato:

| Parametro | Valore attuale | Valore raccomandato |
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

Quasi tutto era sbagliato. Non per errore — per omissione. Nessuno si era preso la briga di configurare il sistema operativo dopo l'installazione.

------------------------------------------------------------------------

## 📦 Huge Pages: il parametro che cambia tutto

Le Huge Pages sono il singolo parametro più impattante per Oracle su Linux. E sono anche quello che viene ignorato più spesso.

### Perché servono

Di default, Linux gestisce la memoria in pagine da 4 KB. Un'SGA da 64 GB significa circa 16,7 milioni di pagine. Ogni pagina ha una entry nella Page Table, e il sistema deve tradurre indirizzi virtuali in fisici per ciascuna. Il TLB (Translation Lookaside Buffer) della CPU può memorizzare solo poche migliaia di traduzioni — il resto viene gestito dalla MMU, che è lenta.

Le Huge Pages sono pagine da 2 MB. La stessa SGA da 64 GB diventa 32.768 pagine. Il TLB regge, la pressione sulla MMU crolla, le performance migliorano.

### Come configurarle

Ho calcolato il numero di Huge Pages necessarie:

``` bash
# SGA = 64 GB = 65536 MB
# Ogni Huge Page = 2 MB
# Pagine necessarie = 65536 / 2 = 32768
# Aggiungo un 1.5% di margine → 33280

echo "vm.nr_hugepages = 33280" >> /etc/sysctl.d/99-oracle.conf
sysctl -p /etc/sysctl.d/99-oracle.conf
```

Verifica:

``` bash
grep -i huge /proc/meminfo
```

Output atteso:

```
HugePages_Total:   33280
HugePages_Free:    33280
HugePages_Rsvd:        0
Hugepagesize:       2048 kB
```

Dopo il riavvio dell'istanza Oracle, la SGA viene allocata nelle Huge Pages:

```
HugePages_Total:   33280
HugePages_Free:      512
HugePages_Rsvd:      480
```

La differenza è misurabile: le `latch free` wait e i `library cache` contention si riducono drasticamente.

------------------------------------------------------------------------

## 🧱 Shared memory e semafori

Oracle usa la shared memory del kernel per la SGA. Se i limiti sono troppo bassi, l'istanza non riesce ad allocare la memoria richiesta — o peggio, frammenta l'allocazione.

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

| Parametro | Significato | Valore |
|---|---|---|
| shmmax | Dimensione massima di un singolo segmento di shared memory | 64 GB |
| shmall | Pagine totali di shared memory allocabili | 64 GB in pagine da 4K |
| shmmni | Numero massimo di segmenti di shared memory | 4096 |
| sem | SEMMSL, SEMMNS, SEMOPM, SEMMNI | 250 32000 100 256 |

Queste impostazioni non sono valori magici. Sono dimensionate sulla SGA del database. Se l'SGA cambia, i parametri vanno ricalcolati.

------------------------------------------------------------------------

## 💾 I/O Scheduler

Il default su RHEL/Oracle Linux 8 con device NVMe è `none` o `mq-deadline`. Per i dischi SAS/SATA tradizionali, il default potrebbe essere `bfq` o `cfq`.

Per Oracle, la raccomandazione è `deadline` (o `mq-deadline` sui kernel più recenti):

``` bash
# Verifica corrente
cat /sys/block/sda/queue/scheduler

# Se non è deadline/mq-deadline, impostarlo
echo "deadline" > /sys/block/sda/queue/scheduler

# Per renderlo permanente via GRUB
grubby --update-kernel=ALL --args="elevator=deadline"
```

`cfq` (Completely Fair Queuing) è pensato per workload desktop — distribuisce equamente l'I/O tra i processi. Ma Oracle non ha bisogno di equità: ha bisogno che le richieste I/O vengano servite nell'ordine che minimizza i seek. `deadline` fa esattamente questo.

------------------------------------------------------------------------

## 🚫 Disabilitare Transparent Huge Pages

Questo è il parametro più insidioso. Le Transparent Huge Pages (THP) sono una funzione del kernel che **sembra** una buona idea: il kernel promuove automaticamente le pagine normali a pagine grandi.

Per Oracle è un disastro. Il processo `khugepaged` lavora in background per compattare le pagine, causando latenze imprevedibili — quei "blocchi di qualche secondo" che il cliente lamentava.

Oracle lo dice esplicitamente nella documentazione: **disabilitare THP**.

``` bash
# Verifica stato corrente
cat /sys/kernel/mm/transparent_hugepage/enabled
# Output tipico: [always] madvise never

# Disabilitare a runtime
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Renderlo permanente via GRUB
grubby --update-kernel=ALL --args="transparent_hugepage=never"
```

Dopo il reboot, verificare:

``` bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# Output atteso: always madvise [never]
```

La differenza è netta: i micro-freeze random scompaiono.

------------------------------------------------------------------------

## 🔒 Security limits

L'utente `oracle` ha bisogno di limiti elevati su file descriptor aperti, processi e memoria bloccabile. I default di Linux sono pensati per utenti interattivi, non per un software che gestisce centinaia di connessioni simultanee.

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

| Limite | Default | Raccomandato | Perché |
|---|---|---|---|
| nofile | 1024 | 65536 | Oracle apre un file descriptor per ogni datafile, redo log, archive log |
| nproc | 4096 | 16384 | Ogni processo Oracle è un processo OS separato |
| memlock | 65536 KB | unlimited | Necessario per il lock della SGA in Huge Pages |
| stack | 8192 KB | 10240-32768 KB | PL/SQL ricorsivo profondo può esaurire lo stack |

Il `memlock unlimited` è fondamentale: senza di esso, Oracle non può bloccare la SGA nelle Huge Pages, rendendo inutile la configurazione fatta prima.

------------------------------------------------------------------------

## ⚡ Swappiness

Il valore di default di `vm.swappiness` è 60. Significa che Linux inizia a swappare quando la pressione sulla memoria è ancora bassa. Per un server database dedicato, questo è inaccettabile: vuoi che la SGA resti in RAM, sempre.

``` bash
echo "vm.swappiness = 1" >> /etc/sysctl.d/99-oracle.conf
sysctl -p /etc/sysctl.d/99-oracle.conf
```

Non zero — uno. Il valore zero disabilita completamente lo swap, il che può causare OOM killer in situazioni di pressione estrema. Il valore uno dice al kernel: "Swappa solo se non c'è davvero più alternativa."

------------------------------------------------------------------------

## 📊 Prima e dopo

Dopo aver applicato tutte le configurazioni e riavviato l'istanza Oracle, ho rifatto le misurazioni.

| Metrica | Prima | Dopo | Variazione |
|---|---|---|---|
| SGA in Huge Pages | No | Sì | — |
| Library cache hit ratio | 92,3% | 99,7% | +7,4% |
| Buffer cache hit ratio | 94,1% | 99,2% | +5,1% |
| Average wait time (db file sequential read) | 8,2 ms | 1,4 ms | -83% |
| Micro-freeze random (>1s) | 5-8 al giorno | 0 | -100% |
| Tempo medio batch mattutino | 47 min | 22 min | -53% |
| CPU utilization media | 78% | 41% | -47% |
| Swap utilizzato | 3,2 GB | 0 MB | -100% |

I numeri parlano da soli. La stessa macchina, lo stesso database, lo stesso carico di lavoro. L'unica differenza: il sistema operativo è stato configurato per fare il suo lavoro.

------------------------------------------------------------------------

## 📋 Checklist finale

Per chi vuole un riepilogo operativo, ecco la checklist completa:

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

Dieci minuti di configurazione. Nessun costo hardware. Nessuna licenza aggiuntiva.

Ma nessuno lo fa, perché il wizard non lo chiede, la documentazione è sepolta in una nota MOS, e il sistema "funziona anche senza." Funziona. Male. E la colpa ricade sempre su Oracle, mai sul fatto che nessuno ha preparato il terreno.

Un database è buono quanto il sistema operativo su cui gira. E un sistema operativo lasciato ai default è un sistema operativo che lavora contro di te.

------------------------------------------------------------------------

## Glossario

**[Huge Pages](/it/glossary/huge-pages/)** — Pagine di memoria da 2 MB (invece dei 4 KB standard) che riducono drasticamente la pressione sulla MMU e sul TLB, migliorando le performance di Oracle su Linux.

**[THP](/it/glossary/thp/)** — Transparent Huge Pages — funzione del kernel Linux che promuove automaticamente le pagine normali a pagine grandi, ma che causa latenze imprevedibili e deve essere disabilitata per Oracle.

**[SGA](/it/glossary/sga/)** — System Global Area — area di memoria condivisa di Oracle Database che contiene buffer cache, shared pool, redo log buffer e altre strutture critiche per le performance.

**[I/O Scheduler](/it/glossary/io-scheduler/)** — Componente del kernel Linux che decide l'ordine in cui le richieste di I/O vengono inviate al disco, con impatto diretto sulle performance del database.

**[Swappiness](/it/glossary/swappiness/)** — Parametro kernel Linux (vm.swappiness) che controlla la propensione del sistema a spostare pagine di memoria nello swap, critico per i server database.
