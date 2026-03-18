---
title: "SGA"
description: "System Global Area — zona de memorie partajată a Oracle Database care conține buffer cache, shared pool, redo log buffer și alte structuri critice pentru performanță."
translationKey: "glossary_sga"
aka: "System Global Area"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

**SGA** (System Global Area) este zona de memorie partajată principală a Oracle Database. Conține structurile de date fundamentale: buffer cache (pagini de date citite de pe disc), shared pool (planuri de execuție și dicționar de date), redo log buffer și large pool.

## Cum funcționează

Dimensiunea SGA este controlată de parametrul `SGA_TARGET` sau `SGA_MAX_SIZE`. Oracle alocă SGA la pornirea instanței în memoria partajată a sistemului de operare. Parametrii kernel Linux `shmmax` și `shmall` trebuie dimensionați pentru a permite alocarea completă a SGA.

## La ce servește

Toată activitatea de citire și scriere a bazei de date trece prin SGA. Un buffer cache eficient evită citirile fizice de pe disc. Un shared pool bine dimensionat evită re-parsing-ul interogărilor. SGA este inima performanțelor Oracle — și trebuie să rezide în Huge Pages pentru a maximiza eficiența.

## De ce contează

O SGA nealocată în Huge Pages înseamnă milioane de intrări în Page Table și overflow constant al TLB. Rezultatul sunt latch free waits, library cache contention și CPU ridicat. Configurarea Huge Pages și a parametrului `memlock unlimited` pentru utilizatorul oracle este prerequisitul pentru orice tuning serios.
