---
title: "PITR"
description: "Point-in-Time Recovery — tehnică de restaurare care permite readucerea unei baze de date la un moment precis în timp, combinând backup-uri și log-uri de tranzacții."
translationKey: "glossary_pitr"
aka: "Point-in-Time Recovery"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**PITR** (Point-in-Time Recovery) este o tehnică de restaurare care permite readucerea unei baze de date la orice moment în timp, nu doar la momentul backup-ului. Se bazează pe combinarea unui backup complet cu log-urile de tranzacții (binary log în MySQL, WAL în PostgreSQL, redo log în Oracle).

## Cum funcționează

Procesul se desfășoară în două faze:

1. **Restaurarea backup-ului**: baza de date este restaurată la ultimul backup disponibil
2. **Replay-ul log-urilor**: log-urile de tranzacții sunt reaplicate de la momentul backup-ului până la momentul dorit, excluzând evenimentul care a cauzat problema

În MySQL, tool-ul `mysqlbinlog` extrage evenimentele din binary log-uri și le reproduce pe baza de date restaurată.

## La ce folosește

PITR este esențial când apare o eroare umană (DROP TABLE, DELETE fără WHERE, UPDATE masiv greșit) și baza de date trebuie restaurată la starea imediat anterioară erorii, fără a pierde orele de lucru dintre ultimul backup și incident.

## Când se folosește

PITR necesită ca binary log-ul să fie activ și ca fișierele binlog să nu fi fost șterse. Retenția binlog-urilor trebuie să acopere cel puțin dublul intervalului dintre două backup-uri consecutive pentru a garanta o acoperire PITR completă.
