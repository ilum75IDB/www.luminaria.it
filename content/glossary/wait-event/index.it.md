---
title: "Wait Event"
description: "Evento di attesa registrato da Oracle ogni volta che una sessione non può procedere e deve attendere una risorsa — I/O, lock, rete o CPU."
translationKey: "glossary_wait_event"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Wait Event** è un indicatore diagnostico di Oracle Database che identifica il motivo per cui una sessione è in attesa anziché lavorare attivamente. Ogni volta che un processo non può procedere — perché attende un blocco dal disco, un lock, una risposta dalla rete o un turno di CPU — Oracle registra un wait event specifico.

## I più comuni

| Wait Event | Significato |
|---|---|
| `db file sequential read` | Lettura singolo blocco — tipica di accessi via indice |
| `db file scattered read` | Lettura multi-blocco — tipica di full table scan |
| `log file sync` | Attesa del commit su redo log |
| `enq: TX - row lock contention` | Conflitto su lock di riga |
| `direct path read` | Lettura diretta (bypass buffer cache) |

## A cosa servono

I wait event sono la base della metodologia diagnostica Oracle. Analizzando quali eventi dominano il DB time (tramite AWR o ASH) si identifica immediatamente la natura del problema: I/O, contention, CPU o rete.

## Dove si trovano

- **In tempo reale**: `V$SESSION_WAIT`, `V$ACTIVE_SESSION_HISTORY`
- **Storici**: report AWR (sezione Top Timed Foreground Events), `DBA_HIST_ACTIVE_SESS_HISTORY`

La regola del DBA: non indovinare cosa rallenta il database — guarda i wait event.
