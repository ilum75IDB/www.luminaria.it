---
title: "PITR"
description: "Point-in-Time Recovery — tecnica di ripristino che permette di riportare un database a un momento preciso nel tempo, combinando backup e log delle transazioni."
translationKey: "glossary_pitr"
aka: "Point-in-Time Recovery"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**PITR** (Point-in-Time Recovery) è una tecnica di ripristino che permette di riportare un database a un qualsiasi momento nel tempo, non solo al momento del backup. Si basa sulla combinazione di un backup completo e dei log delle transazioni (binary log in MySQL, WAL in PostgreSQL, redo log in Oracle).

## Come funziona

Il processo si articola in due fasi:

1. **Restore del backup**: si ripristina il database all'ultimo backup disponibile
2. **Replay dei log**: si riapplicano i log delle transazioni dal momento del backup fino al momento desiderato, escludendo l'evento che ha causato il problema

In MySQL, il tool `mysqlbinlog` estrae gli eventi dai binary log e li riproduce sul database ripristinato.

## A cosa serve

PITR è essenziale quando si verifica un errore umano (DROP TABLE, DELETE senza WHERE, UPDATE massivo sbagliato) e si deve ripristinare il database allo stato immediatamente precedente all'errore, senza perdere le ore di lavoro tra l'ultimo backup e l'incidente.

## Quando si usa

PITR richiede che il binary log sia attivo e che i file binlog non siano stati eliminati. La retention dei binlog deve coprire almeno il doppio dell'intervallo tra due backup consecutivi per garantire una copertura PITR completa.
