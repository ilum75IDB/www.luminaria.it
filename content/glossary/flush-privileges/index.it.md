---
title: "FLUSH PRIVILEGES"
description: "Comando MySQL/MariaDB che ricarica le tabelle dei grant dalla tabella mysql.user, rendendo effettive le modifiche manuali ai privilegi."
translationKey: "glossary_flush-privileges"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

**FLUSH PRIVILEGES** è un comando MySQL/MariaDB che forza il server a ricaricare in memoria le tabelle dei privilegi dal database `mysql`. Rende immediatamente effettive le modifiche ai permessi.

## Come funziona

MySQL mantiene in memoria una cache delle tabelle dei grant (`mysql.user`, `mysql.db`, `mysql.tables_priv`). Quando si usano `CREATE USER` e `GRANT`, MySQL aggiorna sia le tabelle che la cache automaticamente. Ma se si modificano le tabelle dei grant direttamente con `INSERT`, `UPDATE` o `DELETE`, la cache non viene aggiornata. `FLUSH PRIVILEGES` forza il reload della cache dalle tabelle.

## A cosa serve

Il comando è necessario dopo: eliminazione diretta di utenti dalla tabella `mysql.user`, modifiche manuali ai privilegi via DML, o dopo un `DROP USER` di utenti anonimi come parte dell'hardening di sicurezza. Senza il FLUSH, le modifiche non hanno effetto fino al prossimo riavvio del server.

## Quando si usa

Dopo qualsiasi modifica diretta alle tabelle dei grant. Se si usano esclusivamente `CREATE USER`, `GRANT`, `REVOKE` e `DROP USER`, il FLUSH non è tecnicamente necessario perché questi comandi aggiornano la cache automaticamente. Tuttavia, eseguirlo dopo un `DROP USER` di utenti anonimi è una buona pratica per garantire la consistenza.
