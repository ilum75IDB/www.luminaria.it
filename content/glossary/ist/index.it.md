---
title: "IST"
description: "Incremental State Transfer — meccanismo di Galera Cluster per trasferire solo le transazioni mancanti a un nodo che rientra nel cluster."
translationKey: "glossary_ist"
aka: "Incremental State Transfer"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

**IST** (Incremental State Transfer) è il meccanismo con cui un nodo Galera che rientra nel cluster dopo un'assenza breve riceve solo le transazioni mancanti, senza dover scaricare l'intero dataset.

## Come funziona

Quando un nodo si riconnette al cluster, il donatore verifica se le transazioni mancanti sono ancora disponibili nel proprio gcache (Galera cache). Se il gap è coperto dal gcache, viene eseguito un IST: solo le transazioni mancanti vengono inviate al nodo, che le applica e torna in stato Synced. Se il gap supera il gcache, Galera ricade su un SST completo.

## A cosa serve

IST rende il rientro di un nodo nel cluster molto più veloce rispetto a un SST completo. Un nodo che è rimasto offline per qualche minuto o qualche ora può tornare operativo in pochi secondi, senza impatto sulle performance del cluster.

## Quando si usa

IST viene attivato automaticamente quando le condizioni lo permettono. La dimensione del gcache (`gcache.size`) determina quante transazioni il cluster può tenere in memoria per supportare IST. Un gcache più grande permette downtime più lunghi di un nodo senza necessità di SST.
