---
title: "Full Table Scan"
description: "Operazione di lettura in cui Oracle legge tutti i blocchi di una tabella dal primo all'ultimo, senza utilizzare indici."
translationKey: "glossary_full_table_scan"
aka: "TABLE ACCESS FULL"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Full Table Scan** (o TABLE ACCESS FULL) è un'operazione in cui il database legge tutti i blocchi dati di una tabella, dall'inizio alla fine, senza passare per alcun indice.

## Come funziona

Oracle richiede blocchi dal disco (o dalla cache) in sequenza, usando letture multi-blocco (`db file scattered read`). Ogni riga della tabella viene esaminata, indipendentemente dal fatto che soddisfi o meno i criteri della query.

## Quando è un problema

Un full table scan su una tabella grande è spesso il segno di un indice mancante, di statistiche obsolete o di un piano di esecuzione cambiato. Nel report AWR compare come `db file scattered read` nella sezione Top Wait Events, con percentuali elevate di DB time.

## Quando è legittimo

Su tabelle piccole (poche migliaia di righe) o quando la query deve effettivamente leggere la maggior parte dei dati, il full table scan può essere più efficiente di un accesso via indice. Il problema nasce quando Oracle lo sceglie su tabelle con milioni di righe per estrarre pochi record.

## Come si identifica

Nel piano di esecuzione (`EXPLAIN PLAN` o `DBMS_XPLAN`) compare come operazione `TABLE ACCESS FULL`. Nei wait event di AWR/ASH si manifesta come `db file scattered read` dominante.
