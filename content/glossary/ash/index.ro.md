---
title: "ASH"
description: "Active Session History — componenta Oracle care inregistreaza starea fiecarei sesiuni active o data pe secunda, folosita pentru diagnosticarea punctuala a problemelor de performanta."
translationKey: "glossary_ash"
aka: "Active Session History"
articles:
  - "/posts/oracle/oracle-awr-ash"
---

**ASH** (Active Session History) este o componenta a Oracle Database care esantioneaza starea fiecarei sesiuni active o data pe secunda si stocheaza datele intr-un buffer circular in memorie (view-ul `V$ACTIVE_SESSION_HISTORY`).

## Cum functioneaza

In fiecare secunda Oracle inregistreaza pentru fiecare sesiune activa:

- SQL-ul in executie (`SQL_ID`)
- Wait event-ul curent
- Programul si modulul apelant
- Planul de executie utilizat (`SQL_PLAN_HASH_VALUE`)

Datele mai vechi sunt descarcate automat in tabelele AWR (`DBA_HIST_ACTIVE_SESS_HISTORY`) si pastrate pentru perioada configurata.

## La ce serveste

ASH este microscopul DBA-ului: unde AWR arata medii pe intervale orare, ASH permite reconstructia a ceea ce facea o singura sesiune intr-un moment precis. Este instrumentul ideal pentru:

- Identificarea cine executa un SQL problematic
- Intelegerea cand a inceput o problema (la secunda)
- Corelarea sesiunilor, programelor si wait event-urilor in timp real

## Cand se foloseste

Se foloseste cand raportul AWR a identificat deja un SQL sau un wait event dominant si ai nevoie de detalii: ce sesiune, ce program, la ce ora exacta. Regula empirica: **AWR ca sa intelegi ce s-a schimbat, ASH ca sa intelegi de ce**.
