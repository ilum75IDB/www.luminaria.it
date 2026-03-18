---
title: "SQL Injection"
description: "Tecnica di attacco che inserisce codice SQL malevolo negli input di un'applicazione per manipolare le query eseguite dal database, potenzialmente accedendo a dati non autorizzati o compromettendo il sistema."
translationKey: "glossary_sql-injection"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

La **SQL Injection** è una delle vulnerabilità più diffuse e pericolose nelle applicazioni web. Si verifica quando input forniti dall'utente vengono inseriti direttamente nelle query SQL senza validazione o parametrizzazione, permettendo a un attaccante di modificare la logica della query.

## Come funziona

L'attaccante inserisce frammenti di codice SQL nei campi di input dell'applicazione (form di login, campi di ricerca, parametri URL). Se l'applicazione concatena questi input direttamente nelle query SQL, il codice malevolo viene eseguito dal database con i privilegi dell'utente applicativo. In combinazione con il privilegio `FILE` di MySQL e un `secure-file-priv` non configurato, l'attaccante può leggere file di sistema o scrivere file arbitrari sul server.

## A cosa serve

Comprendere la SQL injection è fondamentale per chi gestisce database in produzione, perché molte configurazioni di sicurezza (come `secure-file-priv`, la gestione dei privilegi e la separazione degli utenti) esistono specificamente per mitigare l'impatto di questo tipo di attacco.

## Quando si usa

Il termine descrive un attacco da prevenire, non una tecnica da utilizzare. Le contromisure principali sono: query parametrizzate (prepared statements), validazione degli input, principio del minimo privilegio per gli utenti database, e configurazione corretta di direttive come `secure-file-priv`.
