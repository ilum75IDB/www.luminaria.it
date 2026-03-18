---
title: "Snapshot (Oracle)"
description: "Istantanea delle statistiche di performance catturata periodicamente da AWR e usata per generare report diagnostici comparativi."
translationKey: "glossary_snapshot_oracle"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Snapshot** in Oracle è un'istantanea delle statistiche di performance del database catturata in un preciso momento e conservata nel repository AWR. Di default Oracle genera uno snapshot ogni 60 minuti e li conserva per 8 giorni.

## Come funziona

Ogni snapshot registra centinaia di metriche: wait event, statistiche SQL, metriche di memoria (SGA, PGA), I/O per datafile, statistiche di sistema. Il confronto tra due snapshot genera il report AWR, che mostra cosa è cambiato tra i due istanti.

## Snapshot manuali

In situazioni di emergenza è possibile generare uno snapshot manuale per catturare lo stato corrente:

    EXEC DBMS_WORKLOAD_REPOSITORY.create_snapshot;

Questo è utile quando si vuole un punto di riferimento immediato — ad esempio, prima e dopo un deploy — senza aspettare il ciclo automatico.

## Gestione

Gli snapshot sono consultabili tramite la vista `DBA_HIST_SNAPSHOT`. La retention (quanti giorni conservarli) e l'intervallo (ogni quanti minuti generarli) si configurano con:

    EXEC DBMS_WORKLOAD_REPOSITORY.modify_snapshot_settings(
      retention => 43200,   -- 30 giorni in minuti
      interval  => 30       -- ogni 30 minuti
    );

## Perché sono importanti

Senza snapshot, non c'è AWR. Senza AWR, la diagnosi di un problema di performance diventa un esercizio di intuizione anziché un'analisi basata sui dati. Gli snapshot sono le fondamenta dell'osservabilità in Oracle.
