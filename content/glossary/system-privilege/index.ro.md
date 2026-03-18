---
title: "System Privilege"
description: "Privilegiu Oracle care autorizează operațiuni globale pe baza de date precum CREATE TABLE, CREATE SESSION sau ALTER SYSTEM, independente de orice obiect specific."
translationKey: "glossary_system-privilege"
aka: "Privilegiu de sistem Oracle"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

Un **System Privilege** în Oracle este o autorizare care permite executarea operațiunilor globale pe baza de date, independent de un obiect specific. Exemple tipice includ `CREATE TABLE`, `CREATE SESSION`, `ALTER SYSTEM`, `CREATE USER` și `DROP ANY TABLE`.

## Cum funcționează

Privilegiile de sistem se acordă cu `GRANT` și se revocă cu `REVOKE`. Pot fi atribuite direct unui utilizator sau unui rol. Rolul predefinit `DBA` include peste 200 de privilegii de sistem, motiv pentru care atribuirea sa utilizatorilor aplicativi este o practică periculoasă.

## La ce servește

Privilegiile de sistem definesc ce poate face un utilizator la nivel de bază de date: crea obiecte, gestiona utilizatori, modifica parametri de sistem. Sunt cel mai înalt nivel de autorizare în Oracle și trebuie gestionate cu extremă precauție, urmând principiul privilegiului minim.

## Ce poate merge prost

Un privilegiu de sistem precum `DROP ANY TABLE` permite ștergerea oricărei tabele din orice schemă. Dacă este acordat din greșeală unui utilizator aplicativ, o singură comandă poate distruge date de producție. Distincția între privilegiile de sistem și cele de obiect este fundamentală pentru construirea unui model de securitate robust.
