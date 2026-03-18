---
title: "System Privilege"
description: "Privilegio Oracle che autorizza operazioni globali sul database come CREATE TABLE, CREATE SESSION o ALTER SYSTEM, indipendenti da qualsiasi oggetto specifico."
translationKey: "glossary_system-privilege"
aka: "Privilegio di sistema Oracle"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

Un **System Privilege** in Oracle è un'autorizzazione che permette di eseguire operazioni globali sul database, indipendentemente da uno specifico oggetto. Esempi tipici includono `CREATE TABLE`, `CREATE SESSION`, `ALTER SYSTEM`, `CREATE USER` e `DROP ANY TABLE`.

## Come funziona

I system privileges si assegnano con `GRANT` e si revocano con `REVOKE`. Possono essere assegnati direttamente a un utente o a un ruolo. Il ruolo predefinito `DBA` include oltre 200 system privileges, motivo per cui assegnarlo a utenti applicativi è una pratica pericolosa.

## A cosa serve

I system privileges definiscono cosa un utente può fare a livello di database: creare oggetti, gestire utenti, modificare parametri di sistema. Sono il livello più alto di autorizzazione in Oracle e devono essere gestiti con estrema cautela, seguendo il principio del least privilege.

## Cosa può andare storto

Un system privilege come `DROP ANY TABLE` permette di eliminare qualsiasi tabella di qualsiasi schema. Se assegnato per errore a un utente applicativo, un singolo comando può distruggere dati di produzione. La distinzione tra system privileges e object privileges è fondamentale per costruire un modello di sicurezza robusto.
