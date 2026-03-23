---
title: "Transport Lag"
description: "Intarzierea in transmiterea redo log-urilor de la baza de date primary catre standby intr-o configuratie Data Guard. Indicator critic al sanatatii replicarii."
translationKey: "glossary_transport_lag"
articles:
  - "/posts/oracle/oracle-cloud-migration"
---

**Transport lag** este intarzierea intre momentul in care baza de date primary genereaza un redo log si momentul in care acel redo log este primit de baza de date standby intr-o configuratie Oracle Data Guard. Este unul dintre cei mai importanti indicatori pentru evaluarea sanatatii replicarii.

## Cum se masoara

Transport lag-ul se monitorizeaza printr-un query pe view-ul `V$DATAGUARD_STATS` sau prin Data Guard Broker:

    DGMGRL> SHOW DATABASE 'standby_db' 'TransportLagTarget';

Valoarea este exprimata in format timp (ex. `+00 00:00:03` = 3 secunde intarziere). Un transport lag de cateva secunde este normal in modul Maximum Performance; un lag care creste constant indica o problema de bandwidth sau de redo generate rate.

## Diferenta fata de Apply Lag

| Metrica | Ce masoara |
|---------|-----------|
| **Transport Lag** | Intarzierea in transmiterea redo-ului de la primary la standby |
| **Apply Lag** | Intarzierea in aplicarea redo-ului pe standby dupa receptie |

Transport lag-ul depinde de retea (bandwidth, latenta); apply lag-ul depinde de resursele standby-ului (CPU, I/O). In migrarile cross-site, transport lag-ul este cel mai frecvent bottleneck.

## Impact in migrari

In timpul unei migrari cu Data Guard cross-site, transport lag-ul trebuie monitorizat atent in fazele de incarcare maxima (batch-uri nocturne, varfuri de activitate). Un redo generate rate care depaseste capacitatea VPN-ului produce un transport lag crescator. Inainte de cutover, transport lag-ul trebuie sa fie aproape de zero pentru a garanta ca switchover-ul are loc fara pierdere de date.
