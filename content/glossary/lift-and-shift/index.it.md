---
title: "Lift-and-Shift"
description: "Strategia di migrazione che sposta un sistema da un ambiente a un altro senza modificarne l'architettura, il codice o la configurazione."
translationKey: "glossary_lift-and-shift"
aka: "Rehosting"
articles:
  - "/posts/project-management/tecnica-si-e-yes-and"
---

**Lift-and-Shift** (rehosting) è una strategia di migrazione che consiste nello spostare un sistema da un ambiente all'altro — tipicamente da on-premise a cloud — senza modificarne l'architettura, il codice applicativo o la configurazione. Si prende il sistema così com'è e lo si "solleva e sposta".

## Come funziona

L'infrastruttura viene replicata nell'ambiente di destinazione: stesse macchine virtuali, stessi database, stessi middleware. Il vantaggio è la velocità: non c'è riscrittura del codice, non c'è redesign architetturale. Il rischio è portarsi dietro tutti i problemi dell'ambiente originale, incluse inefficienze e debito tecnico.

## Quando si usa

Quando la priorità è uscire rapidamente da un datacenter (fine contratto, dismissione hardware), quando il budget non permette una rearchitettura, o come prima fase di una migrazione incrementale dove i componenti vengono poi modernizzati uno alla volta.

## Cosa può andare storto

Un lift-and-shift verso il cloud senza ottimizzazione può costare più dell'infrastruttura on-premise originale. Le applicazioni non progettate per il cloud non sfruttano elasticità, auto-scaling e servizi managed. Il risultato è spesso un datacenter privato ricostruito nel cloud a un prezzo superiore.
