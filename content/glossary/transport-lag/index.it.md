---
title: "Transport Lag"
description: "Ritardo nella trasmissione dei redo log dal database primary allo standby in una configurazione Data Guard. Indicatore critico della salute della replica."
translationKey: "glossary_transport_lag"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

Il **transport lag** è il ritardo tra il momento in cui il database primary genera un redo log e il momento in cui quel redo log viene ricevuto dal database standby in una configurazione Oracle Data Guard. È uno degli indicatori più importanti per valutare la salute della replica.

## Come si misura

Il transport lag si monitora con una query sulla vista `V$DATAGUARD_STATS` o tramite Data Guard Broker:

    DGMGRL> SHOW DATABASE 'standby_db' 'TransportLagTarget';

Il valore è espresso in formato tempo (es. `+00 00:00:03` = 3 secondi di ritardo). Un transport lag di pochi secondi è normale in modalità Maximum Performance; un lag che cresce costantemente indica un problema di banda o di redo generate rate.

## Differenza con Apply Lag

| Metrica | Cosa misura |
|---------|-------------|
| **Transport Lag** | Ritardo nella trasmissione dei redo dal primary allo standby |
| **Apply Lag** | Ritardo nell'applicazione dei redo sullo standby dopo la ricezione |

Il transport lag dipende dalla rete (bandwidth, latenza), l'apply lag dipende dalle risorse dello standby (CPU, I/O). Nelle migrazioni cross-site, il transport lag è il collo di bottiglia più comune.

## Impatto nelle migrazioni

Durante una migrazione con Data Guard cross-site, il transport lag va monitorato attentamente durante le fasi di carico massimo (batch notturni, picchi di attività). Un redo generate rate che supera la capacità della VPN produce un transport lag crescente. Prima del cutover, il transport lag deve essere vicino a zero per garantire che lo switchover avvenga senza perdita di dati.
