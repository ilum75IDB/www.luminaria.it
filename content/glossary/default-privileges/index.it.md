---
title: "DEFAULT PRIVILEGES"
description: "Meccanismo PostgreSQL che definisce automaticamente i privilegi da assegnare a tutti gli oggetti futuri creati in uno schema, evitando di dover ripetere i GRANT manualmente."
translationKey: "glossary_default-privileges"
aka: "ALTER DEFAULT PRIVILEGES"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

**DEFAULT PRIVILEGES** è un meccanismo PostgreSQL che permette di definire in anticipo i privilegi che verranno assegnati automaticamente a tutti gli oggetti futuri creati in uno schema. Si configura con il comando `ALTER DEFAULT PRIVILEGES`.

## Come funziona

Il comando `ALTER DEFAULT PRIVILEGES IN SCHEMA schema1 GRANT SELECT ON TABLES TO srv_monitoraggio` fa sì che ogni nuova tabella creata in `schema1` sia automaticamente leggibile da `srv_monitoraggio`. Senza questa configurazione, le tabelle future richiederebbero un GRANT manuale ogni volta.

## A cosa serve

È la parte che la maggior parte degli amministratori dimentica quando crea utenti read-only. I GRANT su `ALL TABLES IN SCHEMA` coprono solo le tabelle esistenti. Le tabelle create dopo richiedono nuovi GRANT — a meno che non si usino i DEFAULT PRIVILEGES. Senza di essi, l'utente di monitoraggio smette di funzionare alla prima nuova tabella.

## Cosa può andare storto

I DEFAULT PRIVILEGES valgono per il ROLE che crea gli oggetti. Se in uno schema più utenti creano tabelle, i default privileges vanno configurati per ciascun creatore. Questo dettaglio causa spesso errori difficili da diagnosticare: "il GRANT c'è, ma la nuova tabella non è leggibile."
