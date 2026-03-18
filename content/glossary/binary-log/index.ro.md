---
title: "Binary log"
description: "Registrul binar secvențial al MySQL care urmărește toate modificările de date, folosit pentru replicare și point-in-time recovery."
translationKey: "glossary_binary-log"
aka: "binlog"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**Binary log-ul** (sau binlog) este un registru secvențial în format binar în care MySQL scrie toate evenimentele care modifică datele: INSERT, UPDATE, DELETE și operațiuni DDL. Fișierele sunt numerotate progresiv (`mysql-bin.000001`, `mysql-bin.000002`, etc.) și gestionate printr-un fișier index.

## Cum funcționează

De la MySQL 8.0, binary log-ul este activat implicit prin parametrul `log_bin`. MySQL creează un nou fișier binlog când serverul pornește, când fișierul curent atinge `max_binlog_size`, sau când se execută `FLUSH BINARY LOGS`. Suportă trei formate de înregistrare: STATEMENT (înregistrează instrucțiunile SQL), ROW (înregistrează modificările la nivel de rând) și MIXED (alegere automată).

## La ce folosește

Binary log-ul are două funcții fundamentale:

- **Replicare**: într-o arhitectură master-slave, slave-ul citește binlog-urile master-ului pentru a replica aceleași operațiuni
- **Point-in-time recovery**: după restaurarea unui backup, binlog-urile permit reaplicarea modificărilor până la un moment precis

## Când se folosește

Binary log-ul este activ implicit pe orice instalare MySQL 8.0+. Gestionarea activă (retenție, purge, monitorizarea spațiului) este necesară pentru a preveni ca fișierele acumulate să umple discul. Comanda `PURGE BINARY LOGS` este modul corect de a elimina fișierele obsolete.
