---
title: "Anonymous User"
description: "Utilizator MySQL/MariaDB fără nume creat automat în timpul instalării. Reprezintă un risc de securitate deoarece poate interfera cu matching-ul utilizatorilor legitimi."
translationKey: "glossary_anonymous-user"
aka: "Utilizator anonim"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

**Anonymous User** (utilizatorul anonim) este un cont MySQL/MariaDB cu username gol (`''@'localhost'`) care este creat automat în timpul instalării. Nu are nume și adesea nu are parolă.

## Cum funcționează

Când un utilizator se conectează, MySQL caută potrivirea cea mai specifică în tabelul `mysql.user`. Utilizatorul anonim `''@'localhost'` este mai specific decât `'mario'@'%'` pentru o conexiune de pe localhost, deoarece `'localhost'` bate `'%'` în ierarhia specificității. În consecință, Mario care se conectează local este autentificat ca utilizator anonim și pierde toate privilegiile sale.

## La ce folosește

Utilizatorul anonim a fost gândit pentru instalări de dezvoltare unde se dorea permiterea conexiunilor fără credențiale. În producție nu are nicio utilitate și reprezintă un risc de securitate: poate captura conexiuni destinate altor utilizatori și acorda acces neautorizat.

## Când se folosește

Niciodată în producție. Prima operație pe orice instalare MySQL/MariaDB de producție este verificarea și eliminarea utilizatorilor anonimi cu `SELECT user, host FROM mysql.user WHERE user = ''` urmat de `DROP USER ''@'localhost'`.
