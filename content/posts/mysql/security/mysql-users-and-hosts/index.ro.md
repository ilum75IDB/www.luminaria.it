---
title: "Utilizatori MySQL: de ce 'mario' și 'mario'@'localhost' nu sunt aceeași persoană"
description: "În MySQL și MariaDB identitatea unui utilizator depinde de host-ul de la care se conectează. Un caz real, modelul de autentificare explicat în profunzime și cele mai frecvente greșeli în gestionarea accesului."
date: "2026-03-08T10:00:00+01:00"
draft: false
translationKey: "mysql_users_and_hosts"
tags: ["MySQL", "MariaDB", "Securitate", "Utilizatori", "Privilegii", "Autentificare"]
categories: ["MySQL", "Security"]
image: "mysql-users-and-hosts.cover.jpg"
---

Acum câteva săptămâni un client mă sună. Ton pragmatic, cerere aparent banală:

> „Trebuie să creez un utilizator pe MySQL pentru o aplicație care trebuie să acceseze o bază de date. Te poți ocupa?"

Sigur. `CREATE USER`, `GRANT`, următorul.

Doar că apoi adaugă: „Aplicația rulează pe două servere diferite. Și uneori ne vom conecta și local pentru mentenanță."

Aici lucrurile nu mai sunt banale. Pentru că în MySQL, a crea „un utilizator" nu înseamnă ceea ce crezi.

---

## Modelul de autentificare MySQL: utilizator + host

Primul lucru de înțeles — și pe care mulți DBA cu experiență în Oracle sau PostgreSQL îl descoperă pe propria piele — este că în MySQL **identitatea unui utilizator nu este doar numele său**.

Este perechea `'utilizator'@'host'`.

Asta înseamnă că:

``` sql
'mario'@'localhost'
'mario'@'192.168.1.10'
'mario'@'%'
```

nu sunt același utilizator. Sunt **trei utilizatori diferiți**. Cu parole diferite, privilegii diferite, comportamente diferite.

Când MySQL primește o conexiune, se uită la două lucruri:
1. Numele de utilizator furnizat
2. Adresa IP (sau hostname-ul) de la care vine conexiunea

Apoi caută în tabelul `mysql.user` rândul care corespunde perechii celei mai specifice. Nu primul găsit. Cel mai specific.

---

## De ce acest model?

Alegerea de design nu este întâmplătoare. MySQL s-a născut în 1995 pentru web. Medii în care aceeași bază de date servește aplicații care rulează pe mașini diferite, rețele diferite, cu nevoi de acces diferite.

Modelul `utilizator@host` permite:

- acordarea accesului complet de pe localhost (pentru DBA)
- acordarea accesului limitat de pe un application server specific
- blocarea a orice altceva

Fără firewall. Fără VPN. Direct în motorul de autentificare.

Este un model puternic. Dar dacă nu îl înțelegi, te mușcă.

---

## Cazul clientului: cum l-am rezolvat

Să revenim la cerere. Aplicația rulează pe două servere (`192.168.1.20` și `192.168.1.21`) și mai este nevoie și de acces local pentru mentenanță.

Tentația este să creezi un singur utilizator cu `'%'` (wildcard = orice host):

``` sql
CREATE USER 'app_vanzari'@'%' IDENTIFIED BY 'ParolaSigura#2026';
GRANT SELECT, INSERT, UPDATE ON vanzari_db.* TO 'app_vanzari'@'%';
```

Funcționează? Da. Este corect? Nu.

Problema cu `'%'` este că acceptă conexiuni de la **orice IP**. Dacă mâine cineva găsește parola, se poate conecta din orice punct al rețelei. Sau al lumii, dacă baza de date este expusă.

Soluția corectă este să creezi **utilizatori specifici pentru fiecare sursă**:

``` sql
-- Acces de pe application server-ul primar
CREATE USER 'app_vanzari'@'192.168.1.20' IDENTIFIED BY 'ParolaSigura#2026';
GRANT SELECT, INSERT, UPDATE ON vanzari_db.* TO 'app_vanzari'@'192.168.1.20';

-- Acces de pe application server-ul secundar
CREATE USER 'app_vanzari'@'192.168.1.21' IDENTIFIED BY 'ParolaSigura#2026';
GRANT SELECT, INSERT, UPDATE ON vanzari_db.* TO 'app_vanzari'@'192.168.1.21';

-- Acces local pentru mentenanță (privilegii diferite)
CREATE USER 'app_vanzari'@'localhost' IDENTIFIED BY 'ParolaMent#2026';
GRANT SELECT ON vanzari_db.* TO 'app_vanzari'@'localhost';
```

Trei utilizatori. Același nume. Privilegii calibrate.

Utilizatorul local are doar `SELECT` pentru că servește la verificări, nu la scrierea datelor. Parolă diferită pentru că contextul de utilizare este diferit.

Principiul privilegiului minim. Aplicat în punctul potrivit.

---

## Capcana matching-ului: cine câștigă?

Aici se nasc majoritatea erorilor.

Dacă există atât `'mario'@'%'` cât și `'mario'@'localhost'`, și Mario se conectează de pe localhost, care utilizator se folosește?

Răspuns: **`'mario'@'localhost'`**.

MySQL sortează rândurile din tabelul `mysql.user` de la cel mai specific la cel mai puțin specific:

1. Host literal exact (`192.168.1.20`)
2. Pattern cu wildcard (`192.168.1.%`)
3. Wildcard total (`%`)

Și folosește **prima potrivire** în ordinea specificității.

Problema clasică este aceasta: creezi `'mario'@'%'` cu toate privilegiile. Apoi cineva creează `'mario'@'localhost'` fără privilegii (sau cu o parolă diferită). Din acel moment, Mario nu mai poate intra de pe local și nimeni nu înțelege de ce.

Am văzut acest scenariu de cel puțin o duzină de ori în producție. Soluția este mereu aceeași: **verifică ce există înainte de a crea**.

``` sql
SELECT user, host, authentication_string
FROM mysql.user
WHERE user = 'mario';
```

Dacă nu o faci înainte, o vei face după. Cu mai multă urgență și mai puțin calm.

---

## MySQL vs MariaDB: diferențele care contează

Modelul `utilizator@host` este identic între MySQL și MariaDB. Dar există diferențe de implementare care merită cunoscute.

**Autentificarea implicită:**

| Versiune | Plugin implicit |
|---|---|
| MySQL 5.7 | `mysql_native_password` |
| MySQL 8.0+ | `caching_sha2_password` |
| MariaDB 10.x | `mysql_native_password` |

Dacă migrezi de la MariaDB la MySQL 8 (sau invers), clienții ar putea să nu se conecteze pentru că plugin-ul de autentificare este diferit. Nu e un bug. E o schimbare de configurație implicită.

**Crearea utilizatorilor:**

În MySQL 8, `GRANT` nu mai creează utilizatori implicit. Trebuie să faci `CREATE USER` mai întâi și `GRANT` după. Întotdeauna.

``` sql
-- MySQL 8: corect
CREATE USER 'app'@'10.0.0.5' IDENTIFIED BY 'pwd123';
GRANT SELECT ON mydb.* TO 'app'@'10.0.0.5';

-- MySQL 5.7 / MariaDB: încă funcționează (dar este depreciat)
GRANT SELECT ON mydb.* TO 'app'@'10.0.0.5' IDENTIFIED BY 'pwd123';
```

Dacă scrii scripturi de provisioning, acest detaliu poate strica o pipeline CI/CD întreagă.

**Roluri:**

MySQL 8.0 a introdus rolurile. MariaDB le suportă din versiunea 10.0.5, dar cu sintaxă ușor diferită.

``` sql
-- MySQL 8.0
CREATE ROLE 'role_citire';
GRANT SELECT ON vanzari_db.* TO 'role_citire';
GRANT 'role_citire' TO 'app_vanzari'@'192.168.1.20';
SET DEFAULT ROLE 'role_citire' FOR 'app_vanzari'@'192.168.1.20';

-- MariaDB 10.x
CREATE ROLE role_citire;
GRANT SELECT ON vanzari_db.* TO role_citire;
GRANT role_citire TO 'app_vanzari'@'192.168.1.20';
SET DEFAULT ROLE role_citire FOR 'app_vanzari'@'192.168.1.20';
```

Diferența pare cosmetică (ghilimele sau nu), dar în scripturi automatizate poate genera erori de sintaxă.

---

## Utilizatorul anonim: fantoma pe care nimeni n-a invitat-o

MySQL vine instalat cu un utilizator anonim: `''@'localhost'`. Fără nume, fără parolă.

Acest utilizator este un artefact istoric al instalărilor de dezvoltare. În producție este un risc de securitate pur.

Utilizatorul anonim câștigă asupra `'mario'@'%'` când conexiunea vine de pe localhost, pentru că `'localhost'` este mai specific decât `'%'`.

Rezultat: Mario se conectează local, MySQL îl autentifică ca utilizator anonim, iar privilegiile lui Mario dispar.

Primul lucru de făcut pe orice instalare MySQL/MariaDB în producție:

``` sql
SELECT user, host FROM mysql.user WHERE user = '';

-- Dacă se găsește:
DROP USER ''@'localhost';
DROP USER ''@'%';  -- dacă există
FLUSH PRIVILEGES;
```

Nu este paranoia. Este igienă.

---

## Checklist operațional

După experiența cu clientul, am formalizat un checklist pe care îl folosesc de fiecare dată când trebuie să creez utilizatori pe MySQL sau MariaDB:

1. **Verifică utilizatorii existenți** cu același nume pe host-uri diferite
2. **Elimină utilizatorii anonimi** dacă sunt prezenți
3. **Creează utilizatori cu host-uri specifice**, niciodată cu `'%'` în producție dacă nu este strict necesar
4. **Acordă doar privilegiile necesare** — `SELECT` dacă e suficient `SELECT`
5. **Folosește `CREATE USER` + `GRANT` separate** (obligatoriu pe MySQL 8)
6. **Verifică plugin-ul de autentificare** dacă clienții au probleme de conexiune
7. **Documentează perechile utilizator/host** — peste șase luni nimeni nu-și va aminti de ce există trei „app_vanzari"

---

## Concluzie

În MySQL și MariaDB un utilizator nu este un nume. Este un nume legat de un punct de origine.

Acest model este puternic pentru că permite segmentarea accesului fără infrastructură suplimentară. Dar este și o sursă de erori subtile dacă nu îl înțelegi în profunzime.

Data viitoare când cineva îți cere „creează un utilizator pe MySQL", înainte de a scrie primul `CREATE USER`, întreabă-te: **de unde se va conecta?**

Răspunsul la această întrebare schimbă totul.
