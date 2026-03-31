---
title: "GTID"
description: "Global Transaction Identifier — identificator unic atribuit fiecărei tranzacții în MySQL pentru a simplifica gestionarea replicării."
translationKey: "glossary_gtid"
aka: "Global Transaction Identifier"
articles:
  - "/posts/mysql/mysql-group-replication-binlog-migration"
  - "/posts/mysql/mysqldump-mysqlpump-mydumper"
---

**GTID** (Global Transaction Identifier) este un identificator unic atribuit automat fiecărei tranzacții confirmate pe un server MySQL. Formatul este `server_uuid:transaction_id` — de exemplu `3E11FA47-71CA-11E1-9E33-C80AA9429562:23`.

## Cum funcționează

Când GTID este activat (`gtid_mode = ON`), fiecare tranzacție primește un identificator care o face trasabilă pe orice server din clusterul de replicare. Replica știe exact ce tranzacții a executat deja și care îi mai lipsesc, fără a fi nevoie să specifice manual pozițiile binlog (fișier + offset).

Setul tuturor GTID-urilor executate pe un server este stocat în variabila `gtid_executed`. Când o replică se conectează la sursă, compară propriul `gtid_executed` cu cel al sursei pentru a determina ce tranzacții lipsesc.

## La ce servește

GTID simplifică radical gestionarea replicării MySQL:

- **Failover automat**: când sursa cade, o replică poate deveni noua sursă și celelalte replici se realiniază automat
- **Verificarea consistenței**: este posibil să verifici dacă două servere au executat exact aceleași tranzacții
- **Backup și restore**: instrumente precum mysqldump și mydumper trebuie să gestioneze corect GTID-urile pentru a evita conflictele de replicare după restore

## Când creează probleme

GTID-urile necesită atenție în timpul operațiunilor de backup și restore. Dacă se restaurează un dump pe un server cu GTID activ fără a configura corect `--set-gtid-purged`, se pot genera conflicte care rup lanțul de replicare.
