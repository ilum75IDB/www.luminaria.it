---
title: "Authentication Plugin"
description: "Modulo MySQL/MariaDB che gestisce il metodo di verifica delle credenziali durante la connessione. Il default cambia tra versioni e può causare problemi di compatibilità."
translationKey: "glossary_authentication-plugin"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

Un **Authentication Plugin** è il modulo che MySQL o MariaDB usa per verificare le credenziali di un utente al momento della connessione. Ogni utente nel sistema è associato a un plugin specifico che determina come la password viene hashata, trasmessa e verificata.

## Come funziona

I plugin principali sono: `mysql_native_password` (default in MySQL 5.7 e MariaDB), che usa un hash SHA1 doppio; `caching_sha2_password` (default in MySQL 8.0+), che usa SHA-256 con caching per migliorare sicurezza e performance. Quando un client si connette, deve supportare il plugin dell'utente a cui sta tentando di autenticarsi.

## A cosa serve

La conoscenza dei plugin di autenticazione è essenziale durante le migrazioni tra versioni o tra MySQL e MariaDB. Un client che supporta solo `mysql_native_password` non riesce a connettersi a un utente con `caching_sha2_password` — e l'errore risultante è spesso criptico e difficile da diagnosticare.

## Quando si usa

Il plugin si specifica al momento della creazione dell'utente (`CREATE USER ... IDENTIFIED WITH <plugin> BY 'password'`) oppure si può verificare e modificare con `ALTER USER`. Quando si scrivono script di provisioning che devono funzionare su versioni diverse di MySQL/MariaDB, è importante specificare esplicitamente il plugin.
