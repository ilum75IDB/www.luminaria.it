---
title: "secure-file-priv"
description: "Directivă de securitate MySQL care limitează directoarele în care serverul poate citi și scrie fișiere, protejând filesystem-ul de operațiuni neautorizate."
translationKey: "glossary_secure-file-priv"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**secure-file-priv** este o variabilă de sistem MySQL care controlează unde instrucțiunile `LOAD DATA INFILE`, `SELECT INTO OUTFILE` și funcția `LOAD_FILE()` pot opera pe filesystem-ul serverului.

## Cum funcționează

Variabila acceptă trei valori: o cale specifică (ex. `/var/lib/mysql-files/`), care limitează operațiunile pe fișiere la acel director; un șir gol (`""`), care nu impune nicio restricție; sau `NULL`, care dezactivează complet operațiunile pe fișiere. Valoarea poate fi setată doar în fișierul de configurare (`my.cnf`) și necesită restart al serviciului pentru a fi modificată — nu poate fi schimbată la runtime.

## La ce folosește

Directiva previne accesul arbitrar la filesystem din partea utilizatorilor MySQL cu privilegiul `FILE`. Fără această protecție, un atacator care exploatează o SQL injection ar putea citi fișiere de sistem (ex. `/etc/passwd`, chei SSH) sau scrie web shell-uri în webroot-ul unui server web de pe același host.

## Când se folosește

`secure-file-priv` trebuie configurată la momentul setup-ului fiecărei instanțe MySQL, specificând un director dedicat. În mediile multi-instanță, fiecare instanță ar trebui să aibă propriul director `secure-file-priv`. Dacă exportul pe fișier e blocat, alternativa recomandată e folosirea clientului mysql din shell cu opțiunile `-B` și `-e` pentru a redirecționa output-ul.
