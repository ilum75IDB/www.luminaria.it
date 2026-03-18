---
title: "ROLE"
description: "Entitatea fundamentală a PostgreSQL care unifică conceptul de utilizator și grup de permisiuni: un ROLE cu LOGIN este un utilizator, fără LOGIN este un container de privilegii."
translationKey: "glossary_postgresql-role"
aka: "Rol PostgreSQL"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

În PostgreSQL, **ROLE** este singura entitate de securitate. Nu există o distincție între "utilizator" și "rol": totul este un ROLE. Un ROLE cu atributul `LOGIN` se comportă ca un utilizator; fără `LOGIN`, este un container de privilegii atribuibil altor ROLE-uri.

## Cum funcționează

`CREATE USER mario` este pur și simplu o scurtătură pentru `CREATE ROLE mario WITH LOGIN`. ROLE-urile pot deține obiecte, moșteni privilegii de la alte ROLE-uri prin atributul `INHERIT`, și fi utilizate pentru a construi ierarhii de permisiuni. Un ROLE "funcțional" (fără LOGIN) grupează privilegii; ROLE-urile "utilizator" (cu LOGIN) le moștenesc.

## La ce servește

Modelul unificat permite proiectarea arhitecturilor de securitate curate: se creează ROLE-uri funcționale precum `role_readonly` sau `role_write`, se atribuie privilegii ROLE-urilor funcționale, apoi se atribuie aceste ROLE-uri utilizatorilor reali. Când sosește un nou coleg, un singur `GRANT role_readonly TO utilizator_nou` este tot ce trebuie.

## De ce contează

Simplitatea modelului este punctul său forte — dar și o capcană dacă nu este înțeles. Mulți administratori atribuie privilegii direct utilizatorilor în loc să folosească ROLE-uri funcționale, creând un ghem de GRANT-uri imposibil de întreținut. Modelul mental corect este: privilegiile merg la ROLE-uri, ROLE-urile merg la utilizatori.
