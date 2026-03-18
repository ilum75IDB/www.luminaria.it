---
title: "Roluri și utilizatori în PostgreSQL: de ce totul este (doar) un ROLE"
description: "PostgreSQL nu distinge între utilizatori și roluri: totul este un ROLE. Modelul mental corect, un caz real și un exemplu complet pentru a construi un utilizator read-only cu adevărat mentenabil."
date: "2026-02-10T10:00:00+01:00"
lastmod: "2026-02-26T09:34:00+01:00"
draft: false
translationKey: "postgresql_roles_and_users"
tags: ["security", "roles", "privileges", "grant", "revoke"]
categories: ["postgresql"]
image: "postgresql_roles_and_users.cover.jpg"
---

Prima dată când am lucrat serios cu PostgreSQL veneam după
ani de experiență cu alte baze de date. Căutam comanda `CREATE USER`. O găseam.
Apoi vedeam `CREATE ROLE`. Apoi `ALTER USER`. Apoi `ALTER ROLE`.\
Pentru câteva minute am gândit: „Bine, aici cineva se distrează
încurcând lumea.”

În realitate, nu. PostgreSQL este mult mai coerent decât pare.
Doar că este coerent în felul lui.

## În PostgreSQL nu există utilizatori. Există roluri.

Cheia este aceasta: **în PostgreSQL totul este un ROLE**.

Un ROLE poate:

-   avea drept de login\
-   să nu aibă drept de login\
-   deține obiecte\
-   moșteni privilegii de la alte roluri\
-   fi utilizat ca un container de privilegii

Ceea ce în alte sisteme numești „utilizator” în PostgreSQL este
pur și simplu un rol cu atributul `LOGIN`.

De fapt:

``` sql
CREATE USER mario;
```

nu este decât un shortcut pentru:

``` sql
CREATE ROLE mario WITH LOGIN;
```

La fel și `ALTER USER`: este doar un alias pentru `ALTER ROLE`.

De ce există, în realitate, doar `CREATE ROLE` și `ALTER ROLE`?\
Pentru că PostgreSQL nu face distincție conceptuală între utilizator și rol.
Este același obiect cu atribute diferite. Minimalist. Elegant. Coerent.

Dacă un rol are `LOGIN`, se comportă ca un utilizator.\
Dacă nu are `LOGIN`, este un container de privilegii.

Când înțelegi cu adevărat acest lucru, modul în care proiectezi securitatea se schimbă.

------------------------------------------------------------------------

## Modelul mental corect

Astăzi gândesc astfel:

-   Creez roluri „funcționale” care reprezintă seturi de privilegii\
-   Atribui aceste roluri utilizatorilor reali\
-   Evit să acord permisiuni direct utilizatorilor

De ce? Pentru că utilizatorii se schimbă. Rolurile nu.

Dacă mâine se alătură un coleg nou, nu rescriu jumătate din grants.\
Îi atribui rolul potrivit și atât.

Arhitectură curată. Fără magie. Fără haos.

------------------------------------------------------------------------

## O poveste reală (fără nume incomode)

Cu ceva timp în urmă mi s-a cerut să creez un utilizator read-only pentru un
sistem de monitorizare.\
Cerere aparent simplă: „Trebuie să citească anumite tabele. Fără scriere.”

Clasicul „e doar read-only”.

Capcana este mereu aceeași: dacă rulezi doar un `GRANT SELECT` pe
tabelele existente, funcționează azi.\
Peste trei luni cineva creează un tabel nou și sistemul de monitorizare începe
să dea erori.\
Și ghici pe cine sună.

Soluția corectă necesită atenție la patru niveluri:

1.  Permisiunea de conectare la bază de date\
2.  Permisiunea de utilizare a schemei (`USAGE`)\
3.  Permisiuni `SELECT` pe tabelele și secvențele existente\
4.  Default privileges pentru obiectele viitoare

Dacă omiți o piesă, mai devreme sau mai târziu plătești prețul.

------------------------------------------------------------------------

## Exemplu: crearea corectă a unui utilizator read-only

Să presupunem că dorim să creăm un utilizator read-only pe două scheme.

Mai întâi creez rolul cu login:

``` sql
CREATE ROLE srv_monitorizare 
WITH LOGIN 
PASSWORD 'ParolaSigura123#';
```

Îl securizez:

``` sql
ALTER ROLE srv_monitorizare 
NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT;
```

Permit conexiunea la baza de date:

``` sql
GRANT CONNECT ON DATABASE mydb TO srv_monitorizare;
```

Permisiune de utilizare a schemelor:

``` sql
GRANT USAGE ON SCHEMA schema1 TO srv_monitorizare;
GRANT USAGE ON SCHEMA schema2 TO srv_monitorizare;
```

Permisiuni de citire pe obiectele existente:

``` sql
GRANT SELECT ON ALL TABLES IN SCHEMA schema1 TO srv_monitorizare;
GRANT SELECT ON ALL TABLES IN SCHEMA schema2 TO srv_monitorizare;

GRANT SELECT ON ALL SEQUENCES IN SCHEMA schema1 TO srv_monitorizare;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA schema2 TO srv_monitorizare;
```

Și acum partea pe care mulți o uită:

``` sql
ALTER DEFAULT PRIVILEGES IN SCHEMA schema1
GRANT SELECT ON TABLES TO srv_monitorizare;

ALTER DEFAULT PRIVILEGES IN SCHEMA schema2
GRANT SELECT ON TABLES TO srv_monitorizare;
```

Astfel, și tabelele viitoare vor fi accesibile pentru citire.

Notă importantă: `ALTER DEFAULT PRIVILEGES` se aplică rolului care
creează obiectele. Dacă mai mulți owners creează tabele în aceleași
scheme, configurația trebuie replicată pentru fiecare dintre ei.

------------------------------------------------------------------------

## De ce acest model este puternic

Faptul că totul este un ROLE îți permite să construiești ierarhii curate.

Exemplu avansat:

``` sql
CREATE ROLE role_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA schema1 TO role_readonly;

CREATE ROLE srv_monitorizare WITH LOGIN PASSWORD '...';
GRANT role_readonly TO srv_monitorizare;
```

Acum pot atribui `role_readonly` la zece utilizatori diferiți fără
a duplica grants.

Asta înseamnă design. Nu doar sintaxă.

------------------------------------------------------------------------

## Concluzie

PostgreSQL nu complică noțiunea de utilizator. O simplifică.\
Există un singur tip de obiect: ROLE. Depinde de noi să-l folosim corect.

Dacă îl tratezi ca pe un simplu „utilizator cu parolă”, funcționează.\
Dacă îl folosești ca pe un bloc arhitectural, devine un instrument puternic
pentru a proiecta o securitate curată, scalabilă și mentenabilă.

Diferența nu este în comenzi.\
Este în modelul mental pe care îl folosești când le aplici.

------------------------------------------------------------------------

## Glosar

**[ROLE](/ro/glossary/postgresql-role/)** — Entitatea fundamentală a PostgreSQL care unifică conceptul de utilizator și grup de permisiuni: un ROLE cu LOGIN este un utilizator, fără LOGIN este un container de privilegii.

**[DEFAULT PRIVILEGES](/ro/glossary/default-privileges/)** — Mecanism PostgreSQL care definește automat privilegiile de atribuit tuturor obiectelor viitoare create într-o schemă, evitând repetarea manuală a GRANT-urilor.

**[Schema](/ro/glossary/schema/)** — Namespace logic în cadrul unei baze de date care grupează tabele, vizualizări, funcții și alte obiecte, permițând organizarea și separarea permisiunilor.

**[GRANT](/ro/glossary/grant/)** — Comandă SQL pentru atribuirea privilegiilor specifice unui utilizator sau rol pe baze de date, tabele sau coloane.

**[Least Privilege](/ro/glossary/least-privilege/)** — Principiu de securitate care prevede atribuirea fiecărui utilizator doar a permisiunilor strict necesare pentru îndeplinirea funcției sale.
