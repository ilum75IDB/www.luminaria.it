---
title: "Least Privilege"
description: "Principio di sicurezza che prevede l'assegnazione a ogni utente o processo solo dei permessi strettamente necessari per svolgere la propria funzione."
translationKey: "glossary_least-privilege"
aka: "Principio del privilegio minimo"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
  - "/posts/oracle/oracle-roles-privileges"
---

Il **Least Privilege** (principio del privilegio minimo) è un principio fondamentale della sicurezza informatica: ogni utente, processo o sistema deve avere solo i permessi strettamente necessari per svolgere la propria funzione, niente di più.

## Come funziona

In ambito database, il principio si applica assegnando privilegi granulari: `SELECT` se l'utente deve solo leggere, `SELECT + INSERT + UPDATE` se deve anche scrivere, mai `ALL PRIVILEGES` se non strettamente necessario. Combinato con il modello `utente@host` di MySQL, il principio può essere applicato anche in base all'origine della connessione.

## A cosa serve

Limitare i privilegi riduce la superficie di attacco. Se un'applicazione viene compromessa, l'attaccante eredita i privilegi dell'utente database dell'applicazione. Se quell'utente ha solo SELECT su un database specifico, il danno è contenuto. Se ha ALL PRIVILEGES, l'intero server è a rischio.

## Quando si usa

Sempre. Il principio del least privilege è applicabile in ogni contesto: utenti database, utenti di sistema operativo, ruoli applicativi, account di servizio. La tentazione di assegnare privilegi ampi "per non avere problemi" è la causa più comune di incidenti di sicurezza evitabili.
