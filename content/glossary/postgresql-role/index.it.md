---
title: "ROLE"
description: "Entità fondamentale di PostgreSQL che unifica il concetto di utente e gruppo di permessi: un ROLE con LOGIN è un utente, senza LOGIN è un contenitore di privilegi."
translationKey: "glossary_postgresql-role"
aka: "Ruolo PostgreSQL"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

In PostgreSQL, **ROLE** è l'unica entità di sicurezza. Non esiste una distinzione tra "utente" e "ruolo": tutto è un ROLE. Un ROLE con l'attributo `LOGIN` si comporta come un utente; senza `LOGIN`, è un contenitore di privilegi assegnabile ad altri ROLE.

## Come funziona

`CREATE USER mario` è semplicemente uno shortcut per `CREATE ROLE mario WITH LOGIN`. I ROLE possono possedere oggetti, ereditare privilegi da altri ROLE tramite l'attributo `INHERIT`, e essere utilizzati per costruire gerarchie di permessi. Un ROLE "funzionale" (senza LOGIN) raggruppa i privilegi; i ROLE "utente" (con LOGIN) li ereditano.

## A cosa serve

Il modello unificato permette di progettare architetture di sicurezza pulite: si creano ROLE funzionali come `role_readonly` o `role_write`, si assegnano i privilegi ai ROLE funzionali, e poi si assegnano questi ROLE agli utenti reali. Quando arriva un nuovo collega, basta un `GRANT role_readonly TO nuovo_utente`.

## Perché è critico

La semplicità del modello è il suo punto di forza — ma anche una trappola se non lo si capisce. Molti amministratori assegnano privilegi direttamente agli utenti invece di usare ROLE funzionali, creando un groviglio di GRANT impossibile da manutenere. Il modello mentale corretto è: i privilegi vanno ai ROLE, i ROLE vanno agli utenti.
