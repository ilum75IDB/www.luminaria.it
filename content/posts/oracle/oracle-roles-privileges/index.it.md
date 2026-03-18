---
title: "Utenti, ruoli e privilegi in Oracle: perché GRANT ALL non è mai la risposta"
description: "Un cliente con tutti gli utenti applicativi connessi come schema owner con ruolo DBA. Come ho ristrutturato il modello di sicurezza Oracle applicando il principio del least privilege — con SQL reale, ruoli custom e Unified Audit."
date: "2026-01-27T10:00:00+01:00"
draft: false
translationKey: "oracle_roles_privileges"
tags: ["security", "roles", "privileges", "grant", "revoke", "audit"]
categories: ["oracle"]
image: "oracle-roles-privileges.cover.jpg"
---

Mi è capitato più volte di entrare in un ambiente Oracle e trovare la stessa situazione: tutti gli utenti applicativi connessi come schema owner, con il ruolo DBA assegnato. Sviluppatori, applicazioni batch, tool di reportistica — tutti con gli stessi privilegi dell'utente che possiede le tabelle.

Quando chiedi perché, la risposta è sempre una variante di: "Così funziona tutto senza problemi di permessi."

Certo. Funziona tutto. Fino al giorno in cui uno sviluppatore lancia un `DROP TABLE` sulla tabella sbagliata. O un batch di import fa un `TRUNCATE` su una tabella di produzione pensando di essere in ambiente di test. O qualcuno esegue un `DELETE FROM clienti` senza la clausola `WHERE`.

Quel giorno il problema non sono più i permessi. È che non hai idea di chi abbia fatto cosa, e non hai nessuno strumento per impedire che succeda di nuovo.

---

## Il contesto: un classico che si ripete

Il cliente era una media azienda con un'applicazione gestionale su Oracle 19c. Circa venti utenti tra sviluppatori, applicativi e operatori. Lo schema applicativo — chiamiamolo `APP_OWNER` — conteneva circa 300 tabelle, una sessantina di viste e qualche decina di procedure PL/SQL.

Il problema era semplice da descrivere:

- Tutti si collegavano come `APP_OWNER`
- `APP_OWNER` aveva il ruolo `DBA`
- Nessun audit configurato
- Nessuna separazione tra chi legge e chi scrive
- Le password erano condivise via email

Non era negligenza. Era inerzia. Il sistema era cresciuto così nel corso degli anni, e nessuno si era mai fermato a ripensare il modello. Funzionava, e questo bastava.

Fino a quando un operatore ha cancellato per errore i dati di fatturazione di un intero trimestre. Nessun log, nessuna traccia, nessun colpevole identificabile. Solo un backup di due giorni prima e un buco nei dati che ha richiesto settimane di lavoro per essere colmato.

---

## Come funziona la sicurezza in Oracle: il modello

Prima di raccontare cosa ho fatto, serve capire come Oracle struttura la sicurezza. Il modello è diverso da PostgreSQL e da MySQL, e le differenze non sono cosmetiche.

### Utente e schema: la stessa cosa (quasi)

In Oracle, **creare un utente significa creare uno schema**. Non sono due concetti separati: l'utente `APP_OWNER` è anche lo schema `APP_OWNER`, e gli oggetti creati da quell'utente vivono in quello schema.

``` sql
CREATE USER app_read IDENTIFIED BY "PasswordSicura#2026"
DEFAULT TABLESPACE users
TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
```

La `QUOTA 0` è intenzionale: questo utente non deve creare oggetti. È un consumatore, non un proprietario.

### Privilegi di sistema vs privilegi di oggetto

Oracle distingue nettamente tra:

- **System privileges**: operazioni globali come `CREATE TABLE`, `CREATE SESSION`, `ALTER SYSTEM`
- **Object privileges**: operazioni su oggetti specifici come `SELECT ON app_owner.clienti`, `EXECUTE ON app_owner.pkg_fatture`

Il ruolo `DBA` include oltre 200 system privilege. Assegnarlo a un utente applicativo è come dare le chiavi dell'intero palazzo a chi deve solo entrare in una stanza.

### I ruoli: predefiniti e custom

Oracle offre ruoli predefiniti (`CONNECT`, `RESOURCE`, `DBA`) e permette di crearne di custom. I ruoli predefiniti hanno un problema storico: `CONNECT` e `RESOURCE` includevano privilegi eccessivi nelle versioni più vecchie. Da Oracle 12c in poi sono stati ridimensionati, ma l'abitudine di assegnarli senza pensarci è dura a morire.

La strada giusta è creare ruoli custom calibrati sulle reali necessità.

---

## L'implementazione: tre ruoli, zero ambiguità

Ho progettato tre ruoli: lettura, scrittura e amministrazione applicativa.

### 1. Ruolo di sola lettura

``` sql
CREATE ROLE app_read_role;

-- Privilegi sulle tabelle
GRANT SELECT ON app_owner.clienti       TO app_read_role;
GRANT SELECT ON app_owner.ordini        TO app_read_role;
GRANT SELECT ON app_owner.fatture       TO app_read_role;
GRANT SELECT ON app_owner.prodotti      TO app_read_role;
GRANT SELECT ON app_owner.movimenti     TO app_read_role;

-- Privilegi sulle viste
GRANT SELECT ON app_owner.v_report_vendite    TO app_read_role;
GRANT SELECT ON app_owner.v_stato_ordini      TO app_read_role;
```

In un ambiente con 300 tabelle non le elenchi una per una a mano. Ho usato un blocco PL/SQL per generare i grant:

``` sql
BEGIN
  FOR t IN (SELECT table_name FROM dba_tables
            WHERE owner = 'APP_OWNER') LOOP
    EXECUTE IMMEDIATE 'GRANT SELECT ON app_owner.'
      || t.table_name || ' TO app_read_role';
  END LOOP;
END;
/
```

Semplice, ripetibile, e soprattutto: documentato. Perché tra sei mesi qualcuno dovrà capire cosa è stato fatto e perché.

### 2. Ruolo di lettura e scrittura

``` sql
CREATE ROLE app_write_role;

-- Eredita tutto dal ruolo di lettura
GRANT app_read_role TO app_write_role;

-- Aggiunge DML sulle tabelle operative
GRANT INSERT, UPDATE, DELETE ON app_owner.ordini    TO app_write_role;
GRANT INSERT, UPDATE, DELETE ON app_owner.movimenti TO app_write_role;
GRANT INSERT, UPDATE ON app_owner.clienti           TO app_write_role;

-- Permesso di esecuzione sulle procedure applicative
GRANT EXECUTE ON app_owner.pkg_ordini   TO app_write_role;
GRANT EXECUTE ON app_owner.pkg_fatture  TO app_write_role;
```

Nota: niente `DELETE` sulla tabella `clienti`. Non perché sia tecnicamente impossibile, ma perché il processo applicativo prevede una disattivazione, non una cancellazione. Il privilegio riflette il processo, non la comodità.

### 3. Ruolo di amministrazione applicativa

``` sql
CREATE ROLE app_admin_role;

-- Eredita il ruolo di scrittura
GRANT app_write_role TO app_admin_role;

-- Aggiunge DDL controllato
GRANT CREATE VIEW TO app_admin_role;
GRANT CREATE PROCEDURE TO app_admin_role;
GRANT CREATE SYNONYM TO app_admin_role;

-- Può gestire le tabelle di configurazione
GRANT INSERT, UPDATE, DELETE ON app_owner.parametri   TO app_admin_role;
GRANT INSERT, UPDATE, DELETE ON app_owner.lookup_tipi TO app_admin_role;
```

Niente `CREATE TABLE`, niente `DROP ANY`, niente `ALTER SYSTEM`. L'admin applicativo gestisce la logica, non la struttura fisica.

---

## Creazione degli utenti e assegnazione

``` sql
-- Utente per i report (sola lettura)
CREATE USER srv_report IDENTIFIED BY "RptSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
GRANT CREATE SESSION TO srv_report;
GRANT app_read_role TO srv_report;

-- Utente applicativo (lettura e scrittura)
CREATE USER srv_app IDENTIFIED BY "AppSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 0 ON users;
GRANT CREATE SESSION TO srv_app;
GRANT app_write_role TO srv_app;

-- DBA applicativo (amministrazione)
CREATE USER dba_app IDENTIFIED BY "DbaSecure#2026"
DEFAULT TABLESPACE users TEMPORARY TABLESPACE temp
QUOTA 10M ON users;
GRANT CREATE SESSION TO dba_app;
GRANT app_admin_role TO dba_app;
```

Ogni utente ha una password propria, un ruolo specifico e una quota disco coerente con il suo scopo. `srv_report` non ha quota perché non deve creare nulla. `dba_app` ha 10 MB perché deve poter creare viste e procedure.

---

## Revoca del ruolo DBA

Il passo più delicato: togliere il `DBA` a `APP_OWNER`.

``` sql
REVOKE DBA FROM app_owner;
```

Una riga. Ma prima di eseguirla, ho verificato che `APP_OWNER` avesse ancora i privilegi necessari per possedere i propri oggetti:

``` sql
SELECT privilege FROM dba_sys_privs WHERE grantee = 'APP_OWNER';
SELECT granted_role FROM dba_role_privs WHERE grantee = 'APP_OWNER';
```

E ho assegnato solo i privilegi strettamente necessari:

``` sql
GRANT CREATE SESSION TO app_owner;
GRANT CREATE TABLE TO app_owner;
GRANT CREATE VIEW TO app_owner;
GRANT CREATE PROCEDURE TO app_owner;
GRANT CREATE SEQUENCE TO app_owner;
GRANT UNLIMITED TABLESPACE TO app_owner;
```

`APP_OWNER` rimane il proprietario degli oggetti, ma non ha più il potere di fare qualsiasi cosa sul database. È un proprietario, non un dio.

---

## Audit: sapere chi fa cosa

Avere i ruoli giusti non basta. Serve sapere chi fa cosa, soprattutto sulle operazioni critiche.

Oracle dalla versione 12c offre **Unified Audit**, che sostituisce il vecchio audit tradizionale con un sistema centralizzato.

``` sql
-- Audit su operazioni DDL critiche
CREATE AUDIT POLICY pol_ddl_critico
ACTIONS CREATE TABLE, DROP TABLE, ALTER TABLE,
        TRUNCATE TABLE, CREATE USER, DROP USER,
        ALTER USER, GRANT, REVOKE;

ALTER AUDIT POLICY pol_ddl_critico ENABLE;

-- Audit su accessi sensibili
CREATE AUDIT POLICY pol_accesso_dati
ACTIONS SELECT ON app_owner.clienti,
        DELETE ON app_owner.fatture,
        UPDATE ON app_owner.fatture;

ALTER AUDIT POLICY pol_accesso_dati ENABLE;

-- Audit sui login falliti
CREATE AUDIT POLICY pol_login_falliti
ACTIONS LOGON;
ALTER AUDIT POLICY pol_login_falliti
ENABLE WHENEVER NOT SUCCESSFUL;
```

Per verificare cosa viene registrato:

``` sql
SELECT * FROM unified_audit_trail
WHERE event_timestamp > SYSDATE - 7
ORDER BY event_timestamp DESC;
```

L'audit non è paranoia. È l'unico modo per rispondere alla domanda "chi ha fatto cosa?" senza dover andare a intuizione.

---

## Il confronto con PostgreSQL e MySQL

Questo articolo è il terzo di una serie sulla gestione della sicurezza nei database relazionali. I primi due coprono [PostgreSQL](/it/posts/postgresql/security/postgresql_roles_and_users/) e [MySQL](/it/posts/mysql/security/mysql-users-and-hosts/).

Le differenze tra i tre sistemi sono sostanziali:

| Aspetto | Oracle | PostgreSQL | MySQL |
|---|---|---|---|
| Utente = schema? | Sì | No (indipendenti) | Sì (database separati) |
| Modello ruoli | Ruoli predefiniti + custom | Tutto è un ROLE | Ruoli da MySQL 8.0 |
| Identità | Nome utente | Nome utente | Coppia utente@host |
| Audit nativo | Unified Audit (12c+) | pgAudit (estensione) | Audit plugin |
| Privilegi granulari | System + Object | Database/Schema/Object | Global/DB/Table/Column |
| GRANT ALL | Esiste ma pericoloso | Esiste, sconsigliato | Esiste, sconsigliato |

In PostgreSQL tutto è un ROLE, e la semplicità del modello è il suo punto di forza. In MySQL l'identità è legata all'host di origine, il che aggiunge un livello di complessità (e di sicurezza) che gli altri non hanno. In Oracle il modello è il più ricco e il più granulare, ma anche il più facile da configurare male per eccesso di opzioni.

Il principio resta lo stesso ovunque: **dai a ciascuno solo quello che gli serve, non un privilegio in più**.

---

## Cosa è cambiato dopo

Il passaggio è stato graduale — due settimane per il rollout completo, con test su ogni applicativo e procedura. Qualche script ha smesso di funzionare perché dava per scontato di avere privilegi che non gli spettavano. Ogni errore era in realtà un problema nascosto che prima era invisibile.

Il risultato:

- **20 utenti nominali** al posto di un unico schema condiviso
- **3 ruoli custom** al posto del ruolo DBA
- **Audit attivo** su DDL e operazioni sensibili
- **Zero incidenti** di cancellazione accidentale nei mesi successivi

Il cliente non ha notato miglioramenti nelle performance. Non era quello l'obiettivo. Ha notato che quando qualcuno sbagliava, il danno era contenuto e tracciabile. E questo, in un ambiente di produzione, vale più di qualsiasi ottimizzazione.

---

## Conclusione

`GRANT ALL PRIVILEGES` e il ruolo `DBA` sono scorciatoie. Funzionano nel senso che eliminano gli errori di permesso. Ma eliminano anche qualsiasi protezione.

La sicurezza in Oracle non è questione di strumenti — gli strumenti ci sono, e sono potenti. È questione di progettazione: decidere chi può fare cosa, documentarlo, implementarlo e poi verificare che funzioni.

Non è il lavoro più glamour del mondo. Ma è quello che fa la differenza tra un database che sopravvive e uno che è davvero sotto controllo.

------------------------------------------------------------------------

## Glossario

**[System Privilege](/it/glossary/system-privilege/)** — Privilegio Oracle che autorizza operazioni globali sul database come CREATE TABLE, CREATE SESSION o ALTER SYSTEM, indipendenti da qualsiasi oggetto specifico.

**[Object Privilege](/it/glossary/object-privilege/)** — Privilegio Oracle che autorizza operazioni su un oggetto specifico del database come SELECT, INSERT o EXECUTE su una tabella, vista o procedura.

**[REVOKE](/it/glossary/revoke/)** — Comando SQL per rimuovere privilegi o ruoli precedentemente assegnati a un utente o ruolo, complementare al comando GRANT.

**[Unified Audit](/it/glossary/unified-audit/)** — Sistema di audit centralizzato introdotto in Oracle 12c che unifica tutti i tipi di audit in un'unica infrastruttura, sostituendo il vecchio audit tradizionale.

**[Least Privilege](/it/glossary/least-privilege/)** — Principio di sicurezza che prevede l'assegnazione a ogni utente solo dei permessi strettamente necessari per svolgere la propria funzione.
