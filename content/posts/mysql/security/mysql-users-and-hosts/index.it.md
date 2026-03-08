---
title: "Utenti MySQL: perché 'mario' e 'mario'@'localhost' non sono la stessa persona"
description: "In MySQL e MariaDB l'identità di un utente dipende dall'host da cui si connette. Un caso reale, il modello di autenticazione spiegato a fondo e gli errori più comuni nella gestione degli accessi."
date: "2026-03-08T10:00:00+01:00"
draft: false
translationKey: "mysql_users_and_hosts"
tags: ["MySQL", "MariaDB", "Security", "Users", "Privileges", "Authentication"]
categories: ["MySQL", "Security"]
image: "mysql-users-and-hosts.cover.jpg"
---

Qualche settimana fa un cliente mi chiama. Tono pragmatico, richiesta apparentemente banale:

> "Devo creare un utente su MySQL per un'applicazione che deve accedere a un database. Puoi occupartene?"

Certo. `CREATE USER`, `GRANT`, avanti il prossimo.

Solo che poi aggiunge: "L'applicazione gira su due server diversi. E a volte ci collegheremo anche da locale per manutenzione."

Ecco. Qui la cosa smette di essere banale. Perché in MySQL, creare "un utente" non significa quello che pensi.

---

## Il modello di autenticazione di MySQL: utente + host

La prima cosa da capire — e che molti DBA con background Oracle o PostgreSQL scoprono a proprie spese — è che in MySQL **l'identità di un utente non è solo il nome**.

È la coppia `'utente'@'host'`.

Questo significa che:

``` sql
'mario'@'localhost'
'mario'@'192.168.1.10'
'mario'@'%'
```

non sono lo stesso utente. Sono **tre utenti diversi**. Con password diverse, privilegi diversi, comportamenti diversi.

Quando MySQL riceve una connessione, guarda due cose:
1. Il nome utente fornito
2. L'indirizzo IP (o hostname) da cui arriva la connessione

Poi cerca nella tabella `mysql.user` la riga che corrisponde alla coppia più specifica. Non la prima trovata. La più specifica.

---

## Perché questo modello?

La scelta progettuale non è casuale. MySQL è nato nel 1995 per il web. Ambienti dove lo stesso database serve applicazioni che girano su macchine diverse, reti diverse, con esigenze di accesso diverse.

Il modello `utente@host` permette di:

- dare accesso completo da localhost (per il DBA)
- dare accesso limitato da un application server specifico
- bloccare tutto il resto

Senza firewall. Senza VPN. Direttamente nel motore di autenticazione.

È un modello potente. Ma se non lo capisci, ti morde.

---

## Il caso del cliente: come l'ho risolto

Torniamo alla richiesta. L'applicazione gira su due server (`192.168.1.20` e `192.168.1.21`) e serve anche un accesso locale per manutenzione.

La tentazione è creare un unico utente con `'%'` (wildcard = qualsiasi host):

``` sql
CREATE USER 'app_vendite'@'%' IDENTIFIED BY 'PasswordSicura#2026';
GRANT SELECT, INSERT, UPDATE ON vendite_db.* TO 'app_vendite'@'%';
```

Funziona? Sì. È corretto? No.

Il problema del `'%'` è che accetta connessioni da **qualsiasi IP**. Se domani qualcuno trova la password, può connettersi da qualunque punto della rete. O del mondo, se il database è esposto.

La soluzione corretta è creare **utenti specifici per ogni sorgente**:

``` sql
-- Accesso dall'application server primario
CREATE USER 'app_vendite'@'192.168.1.20' IDENTIFIED BY 'PasswordSicura#2026';
GRANT SELECT, INSERT, UPDATE ON vendite_db.* TO 'app_vendite'@'192.168.1.20';

-- Accesso dall'application server secondario
CREATE USER 'app_vendite'@'192.168.1.21' IDENTIFIED BY 'PasswordSicura#2026';
GRANT SELECT, INSERT, UPDATE ON vendite_db.* TO 'app_vendite'@'192.168.1.21';

-- Accesso locale per manutenzione (privilegi diversi)
CREATE USER 'app_vendite'@'localhost' IDENTIFIED BY 'PasswordManut#2026';
GRANT SELECT ON vendite_db.* TO 'app_vendite'@'localhost';
```

Tre utenti. Stesso nome. Privilegi calibrati.

L'utente locale ha solo `SELECT` perché serve per verifiche, non per scrivere dati. Password diversa perché il contesto di utilizzo è diverso.

Principio del privilegio minimo. Applicato nel punto giusto.

---

## La trappola del matching: chi vince?

Questo è il punto dove la maggior parte degli errori nasce.

Se esistono sia `'mario'@'%'` che `'mario'@'localhost'`, e Mario si connette da localhost, quale utente viene usato?

Risposta: **`'mario'@'localhost'`**.

MySQL ordina le righe nella tabella `mysql.user` dalla più specifica alla meno specifica:

1. Host letterale esatto (`192.168.1.20`)
2. Pattern con wildcard (`192.168.1.%`)
3. Wildcard totale (`%`)

E usa la **prima corrispondenza** nell'ordine di specificità.

Il problema classico è questo: crei `'mario'@'%'` con tutti i privilegi. Poi qualcuno crea `'mario'@'localhost'` senza privilegi (o con una password diversa). Da quel momento, Mario da locale non riesce più a entrare e nessuno capisce perché.

Ho visto questo scenario almeno una dozzina di volte in produzione. La soluzione è sempre la stessa: **verifica cosa esiste prima di creare**.

``` sql
SELECT user, host, authentication_string
FROM mysql.user
WHERE user = 'mario';
```

Se non lo fai prima, lo farai dopo. Con più urgenza e meno calma.

---

## MySQL vs MariaDB: le differenze che contano

Il modello `utente@host` è identico tra MySQL e MariaDB. Ma ci sono differenze nell'implementazione che vale la pena conoscere.

**Autenticazione di default:**

| Versione | Plugin di default |
|---|---|
| MySQL 5.7 | `mysql_native_password` |
| MySQL 8.0+ | `caching_sha2_password` |
| MariaDB 10.x | `mysql_native_password` |

Se migri da MariaDB a MySQL 8 (o viceversa), i client potrebbero non connettersi perché il plugin di autenticazione è diverso. Non è un bug. È un cambio di default.

**Creazione utenti:**

In MySQL 8, `GRANT` non crea più utenti implicitamente. Devi fare `CREATE USER` prima e `GRANT` dopo. Sempre.

``` sql
-- MySQL 8: corretto
CREATE USER 'app'@'10.0.0.5' IDENTIFIED BY 'pwd123';
GRANT SELECT ON mydb.* TO 'app'@'10.0.0.5';

-- MySQL 5.7 / MariaDB: funziona ancora (ma è deprecato)
GRANT SELECT ON mydb.* TO 'app'@'10.0.0.5' IDENTIFIED BY 'pwd123';
```

Se stai scrivendo script di provisioning, questo dettaglio può rompere una pipeline CI/CD intera.

**Ruoli:**

MySQL 8.0 ha introdotto i ruoli. MariaDB li supporta dalla 10.0.5, ma con sintassi leggermente diversa.

``` sql
-- MySQL 8.0
CREATE ROLE 'role_lettura';
GRANT SELECT ON vendite_db.* TO 'role_lettura';
GRANT 'role_lettura' TO 'app_vendite'@'192.168.1.20';
SET DEFAULT ROLE 'role_lettura' FOR 'app_vendite'@'192.168.1.20';

-- MariaDB 10.x
CREATE ROLE role_lettura;
GRANT SELECT ON vendite_db.* TO role_lettura;
GRANT role_lettura TO 'app_vendite'@'192.168.1.20';
SET DEFAULT ROLE role_lettura FOR 'app_vendite'@'192.168.1.20';
```

La differenza sembra cosmetica (apici o no), ma in script automatizzati può generare errori sintattici.

---

## L'utente anonimo: il fantasma che nessuno invita

MySQL viene installato con un utente anonimo: `''@'localhost'`. Nessun nome, nessuna password.

Questo utente è un residuo storico delle installazioni di sviluppo. In produzione è un rischio di sicurezza puro.

L'utente anonimo vince su `'mario'@'%'` quando la connessione arriva da localhost, perché `'localhost'` è più specifico di `'%'`.

Risultato: Mario si connette da locale, MySQL lo autentica come utente anonimo, e i privilegi di Mario scompaiono.

La prima cosa da fare su qualsiasi installazione MySQL/MariaDB in produzione:

``` sql
SELECT user, host FROM mysql.user WHERE user = '';

-- Se trovato:
DROP USER ''@'localhost';
DROP USER ''@'%';  -- se esiste
FLUSH PRIVILEGES;
```

Non è paranoia. È igiene.

---

## Checklist operativa

Dopo l'esperienza con il cliente, ho formalizzato una checklist che uso ogni volta che devo creare utenti su MySQL o MariaDB:

1. **Verifica utenti esistenti** con lo stesso nome su host diversi
2. **Elimina utenti anonimi** se presenti
3. **Crea utenti con host specifici**, mai con `'%'` in produzione se non strettamente necessario
4. **Assegna solo i privilegi necessari** — `SELECT` se basta `SELECT`
5. **Usa `CREATE USER` + `GRANT` separati** (obbligatorio su MySQL 8)
6. **Verifica il plugin di autenticazione** se i client hanno problemi di connessione
7. **Documenta le coppie utente/host** — tra sei mesi nessuno si ricorderà perché esistono tre "app_vendite"

---

## Conclusione

In MySQL e MariaDB un utente non è un nome. È un nome legato a un punto di origine.

Questo modello è potente perché permette di segmentare gli accessi senza infrastruttura aggiuntiva. Ma è anche una fonte di errori subdoli se non lo si comprende a fondo.

La prossima volta che qualcuno ti chiede "crea un utente su MySQL", prima di scrivere il primo `CREATE USER`, chiediti: **da dove si connetterà?**

La risposta a quella domanda cambia tutto.
