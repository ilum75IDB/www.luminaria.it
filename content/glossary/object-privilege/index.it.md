---
title: "Object Privilege"
description: "Privilegio Oracle che autorizza operazioni su un oggetto specifico del database come SELECT, INSERT, UPDATE o EXECUTE su una tabella, vista o procedura."
translationKey: "glossary_object-privilege"
aka: "Privilegio di oggetto Oracle"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

Un **Object Privilege** in Oracle è un'autorizzazione che permette di eseguire operazioni su un oggetto specifico del database: una tabella, una vista, una sequenza o una procedura PL/SQL. Esempi tipici includono `SELECT ON schema.tabella`, `INSERT ON schema.tabella` e `EXECUTE ON schema.procedura`.

## Come funziona

Gli object privileges si assegnano con `GRANT` specificando l'operazione e l'oggetto target: `GRANT SELECT ON app_owner.clienti TO srv_report`. Possono essere assegnati a singoli utenti o a ruoli. A differenza dei system privileges, operano su un singolo oggetto e non conferiscono poteri globali sul database.

## A cosa serve

Gli object privileges sono lo strumento principale per implementare il principio del least privilege in Oracle. Permettono di costruire modelli di accesso granulari: un utente di reportistica ottiene solo SELECT, un utente applicativo ottiene SELECT + INSERT + UPDATE sulle tabelle operative, e così via. La combinazione con i ruoli custom crea architetture di sicurezza pulite e manutenibili.

## Perché è critico

La differenza tra un `GRANT SELECT ON app_owner.clienti` e un `GRANT DBA` è la differenza tra dare la chiave di una stanza e dare le chiavi dell'intero palazzo. In ambienti con centinaia di tabelle, gli object privileges si gestiscono tipicamente tramite blocchi PL/SQL che generano i grant automaticamente per tutte le tabelle di uno schema.
