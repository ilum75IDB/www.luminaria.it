---
title: "mysqlbinlog"
description: "Utility da riga di comando di MySQL per leggere, filtrare e riapplicare il contenuto dei file binary log."
translationKey: "glossary_mysqlbinlog"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**mysqlbinlog** è l'utility da riga di comando fornita con MySQL per leggere e decodificare il contenuto dei file binary log. È l'unico strumento in grado di convertire il formato binario dei binlog in output leggibile o in istruzioni SQL rieseguibili.

## Come funziona

mysqlbinlog legge i file binlog e produce output in formato testo. Supporta diversi filtri:

- **Per intervallo temporale**: `--start-datetime` e `--stop-datetime` per limitare l'output a una finestra temporale
- **Per database**: `--database` per filtrare gli eventi di un database specifico
- **Per posizione**: `--start-position` e `--stop-position` per selezionare eventi specifici

Con il formato ROW, il flag `--verbose` decodifica le modifiche riga per riga in formato pseudo-SQL commentato, altrimenti l'output è un blob binario illeggibile.

## A cosa serve

mysqlbinlog è utilizzato in due scenari principali:

- **Point-in-time recovery**: estrarre e riapplicare gli eventi dal backup fino al momento desiderato, piping l'output direttamente nel client mysql
- **Debug di replica**: analizzare gli eventi per capire cosa è stato replicato, identificare transazioni problematiche o ricostruire la sequenza di operazioni che ha causato un problema

## Quando si usa

mysqlbinlog è essenziale ogni volta che serve ispezionare cosa è successo nel database dopo un incidente, o quando si deve eseguire un point-in-time recovery. Richiede accesso ai file binlog sul filesystem del server o la possibilità di connettersi al server con `--read-from-remote-server`.
