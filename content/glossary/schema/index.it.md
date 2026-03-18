---
title: "Schema"
description: "Namespace logico all'interno di un database che raggruppa tabelle, viste, funzioni e altri oggetti, permettendo organizzazione e separazione dei permessi."
translationKey: "glossary_schema"
aka: "Schema di database"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

Uno **Schema** in un database relazionale è un namespace logico che raggruppa oggetti come tabelle, viste, funzioni e sequenze. Funziona come un contenitore organizzativo all'interno di un database.

## Come funziona

In PostgreSQL, lo schema predefinito è `public`. Per accedere a un oggetto in un altro schema serve il prefisso: `schema1.tabella`. Il privilegio `USAGE` su uno schema è prerequisito per accedere a qualsiasi oggetto al suo interno — senza `USAGE`, nemmeno un `GRANT SELECT` sulle tabelle funziona.

## A cosa serve

Gli schema permettono di separare logicamente i dati: uno schema per l'applicazione, uno per la reportistica, uno per le tabelle di staging. In Oracle, il concetto è diverso: ogni utente è automaticamente uno schema, e gli oggetti creati dall'utente vivono nel suo schema. In PostgreSQL, schema e utente sono entità indipendenti.

## Perché è critico

La gestione dei permessi sugli schema è la fonte più comune di errori quando si creano utenti con accesso limitato. Dimenticare il `GRANT USAGE ON SCHEMA` è l'errore classico che genera il messaggio "permission denied for schema" anche quando i permessi sulle tabelle sono corretti.
