---
title: "Wait Event"
description: "Eveniment de asteptare inregistrat de Oracle de fiecare data cand o sesiune nu poate continua si trebuie sa astepte o resursa — I/O, lock, retea sau CPU."
translationKey: "glossary_wait_event"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**Wait Event** este un indicator de diagnostic al Oracle Database care identifica motivul pentru care o sesiune asteapta in loc sa lucreze activ. De fiecare data cand un proces nu poate continua — pentru ca asteapta un bloc de pe disc, un lock, un raspuns din retea sau un tur de CPU — Oracle inregistreaza un wait event specific.

## Cele mai comune

| Wait Event | Semnificatie |
|---|---|
| `db file sequential read` | Citire un singur bloc — tipica accesului prin index |
| `db file scattered read` | Citire multi-bloc — tipica full table scan-urilor |
| `log file sync` | Asteptarea commit-ului in redo log |
| `enq: TX - row lock contention` | Conflict de lock pe rand |
| `direct path read` | Citire directa (ocolind buffer cache-ul) |

## La ce servesc

Wait event-urile sunt baza metodologiei de diagnostic Oracle. Analizand ce evenimente domina DB time-ul (prin AWR sau ASH) se identifica imediat natura problemei: I/O, contention, CPU sau retea.

## Unde se gasesc

- **In timp real**: `V$SESSION_WAIT`, `V$ACTIVE_SESSION_HISTORY`
- **Istorice**: rapoarte AWR (sectiunea Top Timed Foreground Events), `DBA_HIST_ACTIVE_SESS_HISTORY`

Regula DBA-ului: nu ghici ce incetineste baza de date — uita-te la wait event-uri.
