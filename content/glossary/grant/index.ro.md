---
title: "GRANT"
description: "Comandă SQL pentru atribuirea de privilegii specifice unui utilizator sau rol pe baze de date, tabele sau coloane. În MySQL 8 nu mai creează utilizatori implicit."
translationKey: "glossary_grant"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
  - "/posts/postgresql/postgresql_roles_and_users"
---

**GRANT** este comanda SQL folosită pentru atribuirea de privilegii unui utilizator sau rol pe obiecte specifice ale bazei de date. În MySQL și MariaDB, privilegiile se atribuie perechii `'utilizator'@'host'`, nu doar numelui de utilizator.

## Cum funcționează

Sintaxa de bază este `GRANT <privilegii> ON <baza_de_date>.<tabel> TO 'utilizator'@'host'`. Privilegiile pot fi granulare (SELECT, INSERT, UPDATE, DELETE) sau globale (ALL PRIVILEGES). În MySQL 8, GRANT nu mai creează utilizatori implicit: e nevoie mai întâi de un `CREATE USER` explicit, apoi GRANT. În MySQL 5.7 și MariaDB, GRANT cu `IDENTIFIED BY` creează utilizatorul și atribuie privilegiile într-o singură comandă.

## La ce folosește

GRANT este mecanismul fundamental pentru implementarea controlului accesului în bazele de date MySQL/MariaDB. Combinat cu modelul `utilizator@host`, permite calibrarea privilegiilor în funcție de originea conexiunii: acces complet de pe localhost pentru DBA, doar citire de pe application server.

## Când se folosește

De fiecare dată când se creează un utilizator sau se modifică permisiunile. Best practice-ul este să atribui întotdeauna privilegiul minim necesar (principiul least privilege) și să folosești `SHOW GRANTS FOR 'utilizator'@'host'` pentru a verifica privilegiile efective.
