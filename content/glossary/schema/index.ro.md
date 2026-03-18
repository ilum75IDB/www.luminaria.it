---
title: "Schema"
description: "Namespace logic în cadrul unei baze de date care grupează tabele, vizualizări, funcții și alte obiecte, permițând organizarea și separarea permisiunilor."
translationKey: "glossary_schema"
aka: "Schemă de bază de date"
articles:
  - "/posts/postgresql/postgresql_roles_and_users"
---

O **Schemă** într-o bază de date relațională este un namespace logic care grupează obiecte precum tabele, vizualizări, funcții și secvențe. Funcționează ca un container organizațional în cadrul unei baze de date.

## Cum funcționează

În PostgreSQL, schema prestabilită este `public`. Pentru a accesa un obiect dintr-o altă schemă este necesar prefixul: `schema1.tabela`. Privilegiul `USAGE` pe o schemă este prerequisit pentru accesarea oricărui obiect din ea — fără `USAGE`, nici măcar un `GRANT SELECT` pe tabele nu funcționează.

## La ce servește

Schemele permit separarea logică a datelor: o schemă pentru aplicație, una pentru raportare, una pentru tabelele de staging. În Oracle, conceptul este diferit: fiecare utilizator este automat o schemă, iar obiectele create de acel utilizator trăiesc în schema sa. În PostgreSQL, schemele și utilizatorii sunt entități independente.

## De ce contează

Gestionarea permisiunilor pe scheme este sursa cea mai comună de erori la crearea utilizatorilor cu acces limitat. Uitarea `GRANT USAGE ON SCHEMA` este greșeala clasică care generează "permission denied for schema" chiar și când permisiunile pe tabele sunt corecte.
