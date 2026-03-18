---
title: "Split-brain"
description: "Condizione critica in un cluster database dove due o più parti operano indipendentemente, accettando scritture divergenti sugli stessi dati."
translationKey: "glossary_split-brain"
articles:
  - "/posts/mysql/galera-cluster-3-nodi"
---

Lo **split-brain** è una condizione critica che si verifica quando un cluster database si divide in due o più partizioni che non possono comunicare tra loro, e ciascuna partizione continua ad accettare scritture indipendentemente. Il risultato sono dati divergenti impossibili da riconciliare automaticamente.

## Come funziona

In un cluster a 3 nodi, se la rete tra il Nodo 1 e i Nodi 2-3 si interrompe, senza protezione dal quorum entrambe le parti potrebbero continuare ad accettare scritture. Quando la rete si ripristina, il cluster si troverebbe con due versioni diverse degli stessi dati. Il meccanismo di quorum previene questo scenario: solo la partizione con la maggioranza dei nodi (quorum) può continuare ad operare.

## A cosa serve

Comprendere lo split-brain è fondamentale per progettare cluster database affidabili. È la ragione principale per cui Galera Cluster richiede un numero dispari di nodi (3, 5, 7) e implementa il meccanismo di quorum. Con un numero pari di nodi, una partizione di rete può dividere il cluster in due metà uguali, nessuna delle quali ha quorum.

## Quando si usa

Il termine split-brain descrive un rischio da evitare, non una funzionalità da attivare. In Galera, la protezione è automatica: i nodi che perdono il quorum passano allo stato Non-Primary e rifiutano le scritture. Il parametro `pc.ignore_quorum` disabilita questa protezione, ma usarlo in produzione è fortemente sconsigliato.
