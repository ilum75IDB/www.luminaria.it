---
title: "Authentication Plugin"
description: "Modul MySQL/MariaDB care gestionează metoda de verificare a credențialelor în timpul conexiunii. Default-ul se schimbă între versiuni și poate cauza probleme de compatibilitate."
translationKey: "glossary_authentication-plugin"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

Un **Authentication Plugin** este modulul pe care MySQL sau MariaDB îl folosește pentru a verifica credențialele unui utilizator la momentul conexiunii. Fiecare utilizator din sistem este asociat cu un plugin specific care determină cum parola este hashată, transmisă și verificată.

## Cum funcționează

Plugin-urile principale sunt: `mysql_native_password` (implicit în MySQL 5.7 și MariaDB), care folosește un hash SHA1 dublu; `caching_sha2_password` (implicit în MySQL 8.0+), care folosește SHA-256 cu caching pentru îmbunătățirea securității și performanței. Când un client se conectează, trebuie să suporte plugin-ul utilizatorului la care încearcă să se autentifice.

## La ce folosește

Cunoașterea plugin-urilor de autentificare este esențială în timpul migrărilor între versiuni sau între MySQL și MariaDB. Un client care suportă doar `mysql_native_password` nu reușește să se conecteze la un utilizator cu `caching_sha2_password` — iar eroarea rezultantă este adesea criptică și greu de diagnosticat.

## Când se folosește

Plugin-ul se specifică la momentul creării utilizatorului (`CREATE USER ... IDENTIFIED WITH <plugin> BY 'password'`) sau poate fi verificat și modificat cu `ALTER USER`. Când se scriu scripturi de provisioning care trebuie să funcționeze pe versiuni diferite de MySQL/MariaDB, este important să se specifice explicit plugin-ul.
