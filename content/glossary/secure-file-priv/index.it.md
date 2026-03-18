---
title: "secure-file-priv"
description: "Direttiva di sicurezza MySQL che limita le directory in cui il server può leggere e scrivere file, proteggendo il filesystem da operazioni non autorizzate."
translationKey: "glossary_secure-file-priv"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**secure-file-priv** è una variabile di sistema MySQL che controlla dove le istruzioni `LOAD DATA INFILE`, `SELECT INTO OUTFILE` e la funzione `LOAD_FILE()` possono operare sul filesystem del server.

## Come funziona

La variabile accetta tre valori: un percorso specifico (es. `/var/lib/mysql-files/`), che limita le operazioni su file a quella directory; una stringa vuota (`""`), che non impone alcuna restrizione; oppure `NULL`, che disabilita completamente le operazioni su file. Il valore è impostabile solo nel file di configurazione (`my.cnf`) e richiede un riavvio del servizio per essere modificato — non è cambiabile a runtime.

## A cosa serve

La direttiva previene l'accesso arbitrario al filesystem da parte di utenti MySQL con il privilegio `FILE`. Senza questa protezione, un attaccante che sfrutta una SQL injection potrebbe leggere file di sistema (es. `/etc/passwd`, chiavi SSH) o scrivere web shell nella webroot di un server web sullo stesso host.

## Quando si usa

`secure-file-priv` va configurata al momento del setup di ogni istanza MySQL, specificando una directory dedicata. In ambienti multi-istanza, ogni istanza dovrebbe avere la propria directory `secure-file-priv`. Se l'export su file è bloccato, l'alternativa consigliata è usare il client mysql da shell con le opzioni `-B` e `-e` per redirigere l'output.
