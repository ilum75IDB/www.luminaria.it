---
title: "Ruoli e utenti in PostgreSQL: perché è tutto (solo) un ROLE"
description: "PostgreSQL non distingue tra utenti e ruoli: tutto è un ROLE. Un modello mentale corretto, un caso reale e un esempio completo per creare un read-only davvero manutenibile."
date: "2026-02-23T09:34:00+01:00"
lastmod: "2026-02-26T09:34:00+01:00"
draft: false
translationKey: "postgresql_roles_and_users"
tags: ["postgresql", "security", "roles", "privileges", "grant", "revoke"]
categories: ["postgresql", "security"]
image: "postgresql_roles_and_users.cover.jpg"
---

La prima volta che ho lavorato seriamente con PostgreSQL arrivavo da
anni di altri database. Cercavo il comando `CREATE USER`. Lo trovavo.
Poi vedevo `CREATE ROLE`. Poi `ALTER USER`. Poi `ALTER ROLE`.\
Per qualche minuto ho pensato: "Ok, qui qualcuno si diverte a confondere
le persone".

In realtà no. PostgreSQL è molto più coerente di quanto sembri. Solo che
lo è a modo suo.

## In PostgreSQL non esistono utenti. Esistono ruoli.

La chiave è questa: **in PostgreSQL tutto è un ROLE**.

Un ROLE può:

-   avere il diritto di login\
-   non avere il diritto di login\
-   possedere oggetti\
-   ereditare permessi da altri ruoli\
-   essere usato come contenitore di privilegi

Quello che in altri database chiami "utente" in PostgreSQL è
semplicemente un ruolo con l'attributo `LOGIN`.

Infatti:

``` sql
CREATE USER mario;
```

non è altro che uno shortcut per:

``` sql
CREATE ROLE mario WITH LOGIN;
```

Stessa cosa per `ALTER USER`: è solo un alias di `ALTER ROLE`.

Perché esiste solo `CREATE ROLE` e `ALTER ROLE`?\
Perché PostgreSQL non distingue concettualmente tra utente e ruolo. È lo
stesso oggetto con attributi diversi. Minimalista, elegante, coerente.

Se un ruolo ha `LOGIN`, si comporta come un utente.\
Se non ha `LOGIN`, è un contenitore di permessi.

Quando lo capisci davvero, cambia il modo in cui progetti la sicurezza.

------------------------------------------------------------------------

## Il modello mentale corretto

Io oggi ragiono così:

-   Creo ruoli "funzionali" che rappresentano insiemi di privilegi\
-   Assegno quei ruoli agli utenti reali\
-   Evito di dare permessi direttamente agli utenti

Perché? Perché gli utenti cambiano. I ruoli no.

Se domani arriva un nuovo collega, non riscrivo mezzo database di
grant.\
Gli assegno il ruolo giusto e fine.

Architettura pulita. Zero magia. Zero caos.

------------------------------------------------------------------------

## Una storia vera (senza nomi imbarazzanti)

Tempo fa mi è stato chiesto di creare un utente di sola lettura per un
sistema di monitoraggio.\
Richiesta apparentemente semplice: "Deve leggere alcune tabelle. Niente
scrittura."

Il classico "tanto è solo read-only".

La trappola è sempre la stessa: se fai solo un `GRANT SELECT` sulle
tabelle esistenti, funziona oggi.\
Tra tre mesi qualcuno crea una nuova tabella e il monitoraggio inizia a
lanciare errori.\
E indovina chi viene chiamato.

La soluzione corretta richiede quattro livelli di attenzione:

1.  Permesso di connessione al database\
2.  Permesso di utilizzo dello schema (`USAGE`)\
3.  Permessi `SELECT` su tabelle e sequence esistenti\
4.  Default privileges per gli oggetti futuri

Se salti un pezzo, prima o poi paghi il conto.

------------------------------------------------------------------------

## Esempio: creazione di un utente read-only fatto bene

Supponiamo di voler creare un utente di sola lettura su due schemi.

Prima creo il ruolo con login:

``` sql
CREATE ROLE srv_monitoraggio 
WITH LOGIN 
PASSWORD 'PasswordSicura123#';
```

Lo metto in sicurezza:

``` sql
ALTER ROLE srv_monitoraggio 
NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT;
```

Permetto la connessione al database:

``` sql
GRANT CONNECT ON DATABASE mydb TO srv_monitoraggio;
```

Permesso di usare gli schemi:

``` sql
GRANT USAGE ON SCHEMA schema1 TO srv_monitoraggio;
GRANT USAGE ON SCHEMA schema2 TO srv_monitoraggio;
```

Permessi di lettura sugli oggetti esistenti:

``` sql
GRANT SELECT ON ALL TABLES IN SCHEMA schema1 TO srv_monitoraggio;
GRANT SELECT ON ALL TABLES IN SCHEMA schema2 TO srv_monitoraggio;

GRANT SELECT ON ALL SEQUENCES IN SCHEMA schema1 TO srv_monitoraggio;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA schema2 TO srv_monitoraggio;
```

E ora la parte che molti dimenticano:

``` sql
ALTER DEFAULT PRIVILEGES IN SCHEMA schema1
GRANT SELECT ON TABLES TO srv_monitoraggio;

ALTER DEFAULT PRIVILEGES IN SCHEMA schema2
GRANT SELECT ON TABLES TO srv_monitoraggio;
```

Così anche le tabelle future saranno leggibili.

Nota importante: gli `ALTER DEFAULT PRIVILEGES` valgono per il ruolo che
crea gli oggetti. Se più owner creano tabelle negli stessi schemi, la
configurazione va replicata per ciascuno.

------------------------------------------------------------------------

## Perché questo modello è potente

Il fatto che tutto sia un ROLE permette di costruire gerarchie pulite.

Esempio evoluto:

``` sql
CREATE ROLE role_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA schema1 TO role_readonly;

CREATE ROLE srv_monitoraggio WITH LOGIN PASSWORD '...';
GRANT role_readonly TO srv_monitoraggio;
```

Ora posso assegnare `role_readonly` a dieci utenti diversi senza
duplicare grant.

Questo è design. Non è solo sintassi.

------------------------------------------------------------------------

## Conclusione

PostgreSQL non complica il concetto di utente. Lo semplifica.\
Esiste solo un tipo di oggetto: il ROLE. Sta a noi usarlo bene.

Se lo tratti come un semplice "utente con password", funziona.\
Se lo usi come mattoncino architetturale, diventa uno strumento potente
per progettare sicurezza pulita, scalabile e manutenibile.

La differenza non sta nei comandi.\
Sta nel modello mentale con cui li usi.
