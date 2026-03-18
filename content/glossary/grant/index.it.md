---
title: "GRANT"
description: "Comando SQL per assegnare privilegi specifici a un utente o ruolo su database, tabelle o colonne. In MySQL 8 non crea più utenti implicitamente."
translationKey: "glossary_grant"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
  - "/posts/postgresql/postgresql_roles_and_users"
---

**GRANT** è il comando SQL usato per assegnare privilegi a un utente o ruolo su oggetti specifici del database. In MySQL e MariaDB, i privilegi vengono assegnati alla coppia `'utente'@'host'`, non solo al nome utente.

## Come funziona

La sintassi base è `GRANT <privilegi> ON <database>.<tabella> TO 'utente'@'host'`. I privilegi possono essere granulari (SELECT, INSERT, UPDATE, DELETE) o globali (ALL PRIVILEGES). In MySQL 8, GRANT non crea più utenti implicitamente: serve prima un `CREATE USER` esplicito, poi il GRANT. In MySQL 5.7 e MariaDB, GRANT con `IDENTIFIED BY` crea l'utente e assegna i privilegi in un solo comando.

## A cosa serve

GRANT è il meccanismo fondamentale per implementare il controllo degli accessi nei database MySQL/MariaDB. Combinato con il modello `utente@host`, permette di calibrare i privilegi in base all'origine della connessione: accesso completo da localhost per il DBA, solo lettura dall'application server.

## Quando si usa

Ogni volta che si crea un utente o si modificano i permessi. La best practice è assegnare sempre il minimo privilegio necessario (principio del least privilege) e usare `SHOW GRANTS FOR 'utente'@'host'` per verificare i privilegi effettivi.
