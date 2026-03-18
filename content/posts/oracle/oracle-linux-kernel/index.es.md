---
title: "Oracle en Linux: los parámetros del kernel que nadie configura"
description: "Un cliente con Oracle 19c en Linux y rendimiento decepcionante. Instalación por defecto, sin tuning. Huge Pages, semáforos, I/O scheduler, THP y límites de seguridad: todo lo que faltaba — con los números del antes y después."
date: "2026-02-24T10:00:00+01:00"
draft: false
translationKey: "oracle_linux_kernel"
tags: ["oracle", "linux", "kernel", "tuning", "hugepages", "performance"]
categories: ["oracle"]
image: "oracle-linux-kernel.cover.jpg"
---

El cliente era una empresa de logística con Oracle 19c Enterprise Edition sobre Oracle Linux 8. Sesenta usuarios concurrentes, un ERP a medida, unos 400 GB de datos. El servidor era un Dell PowerEdge con 128 GB de RAM y 32 cores.

Las quejas eran vagas pero persistentes: "El sistema va lento." "Las queries de la mañana tardan el doble que hace dos meses." "De vez en cuando se congela todo unos segundos."

Cuando entré al servidor, lo primero que verifiqué no fue la base de datos. Fue el sistema operativo.

``` bash
cat /proc/meminfo | grep -i huge
sysctl vm.nr_hugepages
cat /sys/kernel/mm/transparent_hugepage/enabled
```

Resultado: cero Huge Pages configuradas, Transparent Huge Pages activas, parámetros del kernel en sus valores por defecto. La instalación de Oracle se había hecho con el asistente, el sistema operativo no se había tocado.

Ahí estaba el problema. No era Oracle. Era Linux que no había sido preparado para Oracle.

------------------------------------------------------------------------

## 🔍 El diagnóstico

Antes de cambiar nada, medí el estado actual. Se necesitan números, no impresiones.

``` bash
# Estado de la SGA
sqlplus -s / as sysdba <<SQL
SELECT name, value/1024/1024 AS mb
FROM   v$sgainfo
WHERE  name IN ('Maximum SGA Size', 'Free SGA Memory Available');
SQL

# Uso de memoria del sistema
free -h

# Parámetros kernel actuales
sysctl -a | grep -E "kernel.sem|kernel.shm|vm.nr_hugepages|vm.swappiness"

# I/O scheduler en uso
cat /sys/block/sda/queue/scheduler

# Límites del usuario oracle
su - oracle -c "ulimit -a"
```

Esto fue lo que encontré:

| Parámetro | Valor actual | Valor recomendado |
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

Casi todo estaba mal. No por error — por omisión. Nadie se había molestado en configurar el sistema operativo después de la instalación.

------------------------------------------------------------------------

## 📦 Huge Pages: el parámetro que lo cambia todo

Las Huge Pages son el parámetro individual más impactante para Oracle en Linux. Y también son el que se ignora con más frecuencia.

### Por qué importan

Por defecto, Linux gestiona la memoria en páginas de 4 KB. Una SGA de 64 GB significa aproximadamente 16,7 millones de páginas. Cada página tiene una entrada en la Page Table, y el sistema debe traducir direcciones virtuales a físicas para cada una. El TLB (Translation Lookaside Buffer) de la CPU puede almacenar solo unos pocos miles de traducciones — el resto lo gestiona la MMU, que es lenta.

Las Huge Pages son páginas de 2 MB. La misma SGA de 64 GB se convierte en 32.768 páginas. El TLB aguanta, la presión sobre la MMU cae, el rendimiento mejora.

### Cómo configurarlas

Calculé el número de Huge Pages necesarias:

``` bash
# SGA = 64 GB = 65536 MB
# Cada Huge Page = 2 MB
# Páginas necesarias = 65536 / 2 = 32768
# Añado un 1,5% de margen → 33280

echo "vm.nr_hugepages = 33280" >> /etc/sysctl.d/99-oracle.conf
sysctl -p /etc/sysctl.d/99-oracle.conf
```

Verificación:

``` bash
grep -i huge /proc/meminfo
```

Salida esperada:

```
HugePages_Total:   33280
HugePages_Free:    33280
HugePages_Rsvd:        0
Hugepagesize:       2048 kB
```

Después de reiniciar la instancia Oracle, la SGA se asigna en Huge Pages:

```
HugePages_Total:   33280
HugePages_Free:      512
HugePages_Rsvd:      480
```

La diferencia es medible: los `latch free` waits y la `library cache` contention se reducen drásticamente.

------------------------------------------------------------------------

## 🧱 Memoria compartida y semáforos

Oracle usa la memoria compartida del kernel para la SGA. Si los límites son demasiado bajos, la instancia no puede asignar la memoria solicitada — o peor, fragmenta la asignación.

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

| Parámetro | Significado | Valor |
|---|---|---|
| shmmax | Tamaño máximo de un segmento de memoria compartida | 64 GB |
| shmall | Páginas totales de memoria compartida asignables | 64 GB en páginas de 4K |
| shmmni | Número máximo de segmentos de memoria compartida | 4096 |
| sem | SEMMSL, SEMMNS, SEMOPM, SEMMNI | 250 32000 100 256 |

No son valores mágicos. Están dimensionados para la SGA de la base de datos. Si la SGA cambia, los parámetros deben recalcularse.

------------------------------------------------------------------------

## 💾 I/O Scheduler

El default en RHEL/Oracle Linux 8 con dispositivos NVMe es `none` o `mq-deadline`. Para discos SAS/SATA tradicionales, el default puede ser `bfq` o `cfq`.

Para Oracle, la recomendación es `deadline` (o `mq-deadline` en kernels más recientes):

``` bash
# Verificar configuración actual
cat /sys/block/sda/queue/scheduler

# Si no es deadline/mq-deadline, configurarlo
echo "deadline" > /sys/block/sda/queue/scheduler

# Hacerlo permanente vía GRUB
grubby --update-kernel=ALL --args="elevator=deadline"
```

`cfq` (Completely Fair Queuing) está diseñado para cargas de trabajo de escritorio — distribuye el I/O equitativamente entre procesos. Pero Oracle no necesita equidad: necesita que las solicitudes de I/O se atiendan en el orden que minimiza los seeks. `deadline` hace exactamente eso.

------------------------------------------------------------------------

## 🚫 Deshabilitar Transparent Huge Pages

Este es el parámetro más insidioso. Las Transparent Huge Pages (THP) son una función del kernel que **parece** buena idea: el kernel promueve automáticamente las páginas normales a páginas grandes.

Para Oracle es un desastre. El proceso `khugepaged` trabaja en segundo plano para compactar páginas, causando picos de latencia impredecibles — esos "bloqueos de unos segundos" que el cliente reportaba.

Oracle lo dice explícitamente en la documentación: **deshabilitar THP**.

``` bash
# Verificar estado actual
cat /sys/kernel/mm/transparent_hugepage/enabled
# Salida típica: [always] madvise never

# Deshabilitar en tiempo de ejecución
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Hacerlo permanente vía GRUB
grubby --update-kernel=ALL --args="transparent_hugepage=never"
```

Después del reinicio, verificar:

``` bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# Salida esperada: always madvise [never]
```

La diferencia es nítida: los micro-bloqueos aleatorios desaparecen.

------------------------------------------------------------------------

## 🔒 Límites de seguridad

El usuario `oracle` necesita límites elevados en descriptores de archivo abiertos, procesos y memoria bloqueable. Los defaults de Linux están pensados para usuarios interactivos, no para software que gestiona cientos de conexiones simultáneas.

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

| Límite | Default | Recomendado | Por qué |
|---|---|---|---|
| nofile | 1024 | 65536 | Oracle abre un descriptor por cada datafile, redo log, archive log |
| nproc | 4096 | 16384 | Cada proceso Oracle es un proceso OS separado |
| memlock | 65536 KB | unlimited | Necesario para bloquear la SGA en Huge Pages |
| stack | 8192 KB | 10240-32768 KB | PL/SQL recursivo profundo puede agotar el stack |

El `memlock unlimited` es fundamental: sin él, Oracle no puede bloquear la SGA en Huge Pages, haciendo inútil la configuración anterior.

------------------------------------------------------------------------

## ⚡ Swappiness

El valor por defecto de `vm.swappiness` es 60. Significa que Linux empieza a hacer swap cuando la presión sobre la memoria es aún baja. Para un servidor de base de datos dedicado, esto es inaceptable: quieres que la SGA permanezca en RAM, siempre.

``` bash
echo "vm.swappiness = 1" >> /etc/sysctl.d/99-oracle.conf
sysctl -p /etc/sysctl.d/99-oracle.conf
```

No cero — uno. Un valor de cero deshabilita completamente el swap, lo que puede provocar el OOM killer en situaciones de presión extrema. Un valor de uno le dice al kernel: "Solo haz swap cuando realmente no haya alternativa."

------------------------------------------------------------------------

## 📊 Antes y después

Después de aplicar todas las configuraciones y reiniciar la instancia Oracle, repetí las mediciones.

| Métrica | Antes | Después | Variación |
|---|---|---|---|
| SGA en Huge Pages | No | Sí | — |
| Library cache hit ratio | 92,3% | 99,7% | +7,4% |
| Buffer cache hit ratio | 94,1% | 99,2% | +5,1% |
| Tiempo medio de espera (db file sequential read) | 8,2 ms | 1,4 ms | -83% |
| Micro-bloqueos aleatorios (>1s) | 5-8 al día | 0 | -100% |
| Tiempo medio batch matutino | 47 min | 22 min | -53% |
| Utilización CPU media | 78% | 41% | -47% |
| Swap utilizado | 3,2 GB | 0 MB | -100% |

Los números hablan por sí mismos. La misma máquina, la misma base de datos, la misma carga de trabajo. La única diferencia: el sistema operativo fue configurado para hacer su trabajo.

------------------------------------------------------------------------

## 📋 Checklist final

Para quien quiera un resumen operativo, aquí está la checklist completa:

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

Diez minutos de configuración. Sin coste de hardware. Sin licencias adicionales.

Pero nadie lo hace, porque el asistente no lo pide, la documentación está enterrada en una nota MOS, y el sistema "funciona sin ello." Funciona. Mal. Y la culpa siempre recae en Oracle, nunca en que nadie preparó el terreno.

Una base de datos es tan buena como el sistema operativo sobre el que corre. Y un sistema operativo dejado en sus valores por defecto es un sistema operativo que trabaja en tu contra.

------------------------------------------------------------------------

## Glosario

**[Huge Pages](/es/glossary/huge-pages/)** — Páginas de memoria de 2 MB (en lugar de los 4 KB estándar) que reducen drásticamente la presión sobre la MMU y el TLB, mejorando el rendimiento de Oracle en Linux.

**[THP](/es/glossary/thp/)** — Transparent Huge Pages — función del kernel Linux que promueve automáticamente las páginas normales a páginas grandes, pero que causa latencias impredecibles y debe deshabilitarse para Oracle.

**[SGA](/es/glossary/sga/)** — System Global Area — área de memoria compartida de Oracle Database que contiene buffer cache, shared pool, redo log buffer y otras estructuras críticas para el rendimiento.

**[I/O Scheduler](/es/glossary/io-scheduler/)** — Componente del kernel Linux que decide el orden en que las solicitudes de I/O se envían al disco, con impacto directo en el rendimiento de la base de datos.

**[Swappiness](/es/glossary/swappiness/)** — Parámetro del kernel Linux (vm.swappiness) que controla la propensión del sistema a mover páginas de memoria al swap, crítico para servidores de base de datos.
