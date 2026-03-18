---
title: "ASH"
description: "Active Session History — componente Oracle che registra lo stato di ogni sessione attiva una volta al secondo, usato per la diagnosi puntuale dei problemi di performance."
translationKey: "glossary_ash"
aka: "Active Session History"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**ASH** (Active Session History) è un componente di Oracle Database che campiona lo stato di ogni sessione attiva una volta al secondo e conserva i dati in un buffer circolare in memoria (vista `V$ACTIVE_SESSION_HISTORY`).

## Come funziona

Ogni secondo Oracle registra per ogni sessione attiva:

- SQL in esecuzione (`SQL_ID`)
- Wait event corrente
- Programma e modulo chiamante
- Piano di esecuzione utilizzato (`SQL_PLAN_HASH_VALUE`)

I dati più vecchi vengono scaricati automaticamente nelle tabelle AWR (`DBA_HIST_ACTIVE_SESS_HISTORY`) e conservati per il periodo configurato.

## A cosa serve

ASH è il microscopio del DBA: dove AWR mostra medie su intervalli orari, ASH permette di ricostruire cosa stava facendo una singola sessione in un preciso istante. È lo strumento ideale per:

- Identificare chi sta eseguendo un SQL problematico
- Capire quando un problema è iniziato (al secondo)
- Correlare sessioni, programmi e wait event in tempo reale

## Quando si usa

Si usa quando il report AWR ha già identificato un SQL o un wait event dominante e si ha bisogno di dettaglio: quale sessione, quale programma, a che ora esatta. La regola empirica: **AWR per capire cosa è cambiato, ASH per capire perché**.
