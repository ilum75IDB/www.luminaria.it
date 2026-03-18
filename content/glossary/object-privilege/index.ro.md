---
title: "Object Privilege"
description: "Privilegiu Oracle care autorizează operațiuni pe un obiect specific al bazei de date precum SELECT, INSERT, UPDATE sau EXECUTE pe o tabelă, vizualizare sau procedură."
translationKey: "glossary_object-privilege"
aka: "Privilegiu de obiect Oracle"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

Un **Object Privilege** în Oracle este o autorizare care permite executarea operațiunilor pe un obiect specific al bazei de date: o tabelă, o vizualizare, o secvență sau o procedură PL/SQL. Exemple tipice includ `SELECT ON schema.tabela`, `INSERT ON schema.tabela` și `EXECUTE ON schema.procedura`.

## Cum funcționează

Privilegiile de obiect se acordă cu `GRANT` specificând operațiunea și obiectul țintă: `GRANT SELECT ON app_owner.clienti TO srv_report`. Pot fi atribuite utilizatorilor individuali sau rolurilor. Spre deosebire de privilegiile de sistem, operează pe un singur obiect și nu conferă puteri globale asupra bazei de date.

## La ce servește

Privilegiile de obiect sunt instrumentul principal pentru implementarea principiului privilegiului minim în Oracle. Permit construirea modelelor de acces granulare: un utilizator de raportare primește doar SELECT, un utilizator aplicativ primește SELECT + INSERT + UPDATE pe tabelele operaționale, și așa mai departe. Combinate cu roluri personalizate, creează arhitecturi de securitate curate și ușor de întreținut.

## De ce contează

Diferența între un `GRANT SELECT ON app_owner.clienti` și un `GRANT DBA` este diferența între a da cheia unei camere și a da cheile întregii clădiri. În medii cu sute de tabele, privilegiile de obiect se gestionează de obicei prin blocuri PL/SQL care generează automat grant-urile pentru toate tabelele dintr-o schemă.
