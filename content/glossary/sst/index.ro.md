---
title: "SST"
description: "State Snapshot Transfer — mecanism Galera Cluster pentru transferul unei copii complete a datelor către un nod care se alătură clusterului."
translationKey: "glossary_sst"
aka: "State Snapshot Transfer"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**SST** (State Snapshot Transfer) este mecanismul prin care un nod Galera care se alătură clusterului (sau care a fost offline prea mult timp) primește o copie completă a întregului set de date de la un nod donator.

## Cum funcționează

Când un nod se alătură clusterului și gap-ul de tranzacții lipsă depășește dimensiunea gcache-ului, clusterul inițiază un SST. Nodul donator creează un snapshot complet al bazei de date și îl transferă nodului receptor. Metodele disponibile sunt: `mariabackup` (nu blochează donatorul), `rsync` (rapid dar blochează donatorul pe citire), și `mysqldump` (lent și blocant).

## La ce folosește

SST este esențial pentru două scenarii: adăugarea unui nod nou la cluster (primul join) și recuperarea unui nod care a fost offline atât de mult timp încât tranzacțiile lipsă nu mai sunt disponibile în gcache-ul donatorului.

## Când se folosește

SST este declanșat automat de Galera când este necesar. Alegerea metodei SST (`wsrep_sst_method`) se face în faza de configurare. În producție, `mariabackup` este alegerea recomandată deoarece nu blochează nodul donator, evitând degradarea clusterului în timpul transferului.
