---
title: "RAC"
description: "Real Application Clusters — tecnologia Oracle che permette a più istanze di accedere contemporaneamente allo stesso database, garantendo alta disponibilità e scalabilità."
translationKey: "glossary_rac"
aka: "Real Application Clusters"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**RAC** (Real Application Clusters) è la tecnologia Oracle che consente a più istanze del database di accedere contemporaneamente allo stesso storage condiviso. Se un nodo si guasta, gli altri continuano a servire le richieste senza interruzione — il failover è trasparente per le applicazioni.

## Come funziona

Un cluster RAC è composto da due o più server (nodi) collegati tra loro da una rete privata ad alta velocità (interconnect) e da uno storage condiviso (tipicamente ASM — Automatic Storage Management). Ogni nodo esegue la propria istanza Oracle, ma tutti accedono agli stessi datafile.

Il protocollo **Cache Fusion** gestisce la coerenza dei dati tra i nodi: quando un blocco modificato da un nodo serve a un altro, viene trasferito direttamente via interconnect senza passare dal disco.

## Alta disponibilità

Se un nodo va giù, le sessioni attive vengono automaticamente trasferite ai nodi rimanenti tramite **TAF** (Transparent Application Failover) o **Application Continuity**. Il tempo di failover dipende dalla configurazione, ma tipicamente è dell'ordine di secondi.

## Implicazioni di licensing

RAC è un'opzione della Enterprise Edition e ha un costo di licensing significativo. In fase di migrazione cloud, il conteggio delle licenze RAC è uno degli aspetti più delicati: su OCI con BYOL le licenze on-premises vengono riutilizzate, su altri cloud provider il costo può moltiplicarsi.

## Quando serve davvero

RAC è giustificato quando servono alta disponibilità con failover automatico e scalabilità orizzontale. Per ambienti con pochi utenti o requisiti di uptime standard, un singolo nodo con Data Guard è spesso una soluzione più semplice e meno costosa.
