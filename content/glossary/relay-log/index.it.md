---
title: "Relay log"
description: "File di log intermedio sullo slave MySQL che riceve gli eventi dal binary log del master prima che vengano eseguiti localmente."
translationKey: "glossary_relay-log"
articles:
  - "/posts/mysql/binary-log-mysql"
---

Il **relay log** è un file di log intermedio presente sullo slave in un'architettura di replica MySQL. Contiene gli eventi ricevuti dal binary log del master, in attesa di essere eseguiti localmente dal thread SQL dello slave.

## Come funziona

Il flusso della replica MySQL passa attraverso il relay log in tre fasi:

1. L'**I/O thread** dello slave si connette al master e legge i binary log
2. Gli eventi ricevuti vengono scritti nel relay log locale
3. Il **SQL thread** dello slave legge gli eventi dal relay log e li esegue sul database locale

Questa architettura a due thread permette di disaccoppiare la ricezione dei dati dalla loro applicazione: lo slave può continuare a ricevere eventi dal master anche se l'esecuzione locale è temporaneamente rallentata.

## A cosa serve

Il relay log è il meccanismo che garantisce la consistenza della replica. Funge da buffer tra il master e l'applicazione locale degli eventi, permettendo allo slave di gestire eventuali differenze di velocità senza perdere dati.

## Quando si usa

Il relay log viene creato automaticamente quando si configura una replica MySQL. Non richiede gestione manuale diretta, ma il suo stato (posizione corrente, eventuale ritardo) è visibile tramite `SHOW REPLICA STATUS` ed è fondamentale per diagnosticare problemi di replica lag.
