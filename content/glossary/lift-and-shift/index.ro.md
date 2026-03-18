---
title: "Lift-and-Shift"
description: "Strategie de migrare care mută un sistem dintr-un mediu în altul fără a-i modifica arhitectura, codul sau configurația."
translationKey: "glossary_lift-and-shift"
aka: "Rehosting"
articles:
  - "/posts/project-management/tecnica-si-e-yes-and"
---

**Lift-and-Shift** (rehosting) este o strategie de migrare care constă în mutarea unui sistem dintr-un mediu în altul — de obicei din on-premise în cloud — fără a-i modifica arhitectura, codul aplicativ sau configurația. Sistemul este luat ca atare și "ridicat și mutat".

## Cum funcționează

Infrastructura este replicată în mediul de destinație: aceleași mașini virtuale, aceleași baze de date, același middleware. Avantajul este viteza: nu există rescrierea codului, nu există reproiectare arhitecturală. Riscul este de a duce cu sine toate problemele din mediul original, inclusiv ineficiențele și datoria tehnică.

## Când se folosește

Când prioritatea este ieșirea rapidă dintr-un datacenter (expirare contract, dezafectare hardware), când bugetul nu permite o rearchitectură, sau ca primă fază a unei migrări incrementale unde componentele sunt apoi modernizate una câte una.

## Ce poate merge prost

Un lift-and-shift către cloud fără optimizare poate costa mai mult decât infrastructura on-premise originală. Aplicațiile neproiectate pentru cloud nu profită de elasticitate, auto-scaling și servicii gestionate. Rezultatul este adesea un datacenter privat reconstruit în cloud la un preț superior.
