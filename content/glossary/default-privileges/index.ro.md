---
title: "DEFAULT PRIVILEGES"
description: "Mecanism PostgreSQL care definește automat privilegiile de atribuit tuturor obiectelor viitoare create într-o schemă, evitând repetarea manuală a GRANT-urilor."
translationKey: "glossary_default-privileges"
aka: "ALTER DEFAULT PRIVILEGES"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

**DEFAULT PRIVILEGES** este un mecanism PostgreSQL care permite definirea în avans a privilegiilor ce vor fi atribuite automat tuturor obiectelor viitoare create într-o schemă. Se configurează cu comanda `ALTER DEFAULT PRIVILEGES`.

## Cum funcționează

Comanda `ALTER DEFAULT PRIVILEGES IN SCHEMA schema1 GRANT SELECT ON TABLES TO srv_monitorizare` face ca fiecare tabelă nouă creată în `schema1` să fie automat citibilă de `srv_monitorizare`. Fără această configurare, tabelele viitoare ar necesita un GRANT manual de fiecare dată.

## La ce servește

Este partea pe care majoritatea administratorilor o uită când creează utilizatori read-only. GRANT-urile pe `ALL TABLES IN SCHEMA` acoperă doar tabelele existente. Tabelele create ulterior necesită noi GRANT-uri — doar dacă nu se folosesc DEFAULT PRIVILEGES. Fără ele, utilizatorul de monitorizare încetează să funcționeze la prima tabelă nouă.

## Ce poate merge prost

DEFAULT PRIVILEGES se aplică ROLE-ului care creează obiectele. Dacă într-o schemă mai mulți utilizatori creează tabele, default privileges trebuie configurate pentru fiecare creator. Acest detaliu cauzează adesea erori greu de diagnosticat: "GRANT-ul există, dar tabela nouă nu este citibilă."
