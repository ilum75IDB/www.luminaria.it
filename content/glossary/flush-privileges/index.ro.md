---
title: "FLUSH PRIVILEGES"
description: "Comandă MySQL/MariaDB care reîncarcă tabelele de grant din mysql.user, făcând efective modificările manuale ale privilegiilor."
translationKey: "glossary_flush-privileges"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

**FLUSH PRIVILEGES** este o comandă MySQL/MariaDB care forțează serverul să reîncarce în memorie tabelele de privilegii din baza de date `mysql`. Face imediat efective modificările permisiunilor.

## Cum funcționează

MySQL menține în memorie un cache al tabelelor de grant (`mysql.user`, `mysql.db`, `mysql.tables_priv`). Când se folosesc `CREATE USER` și `GRANT`, MySQL actualizează atât tabelele cât și cache-ul automat. Dar dacă tabelele de grant sunt modificate direct cu `INSERT`, `UPDATE` sau `DELETE`, cache-ul nu se actualizează. `FLUSH PRIVILEGES` forțează reîncărcarea cache-ului din tabele.

## La ce folosește

Comanda este necesară după: ștergerea directă a utilizatorilor din tabelul `mysql.user`, modificări manuale ale privilegiilor prin DML, sau după un `DROP USER` al utilizatorilor anonimi ca parte a hardening-ului de securitate. Fără FLUSH, modificările nu au efect până la următorul restart al serverului.

## Când se folosește

După orice modificare directă a tabelelor de grant. Dacă se folosesc exclusiv `CREATE USER`, `GRANT`, `REVOKE` și `DROP USER`, FLUSH nu este tehnic necesar deoarece aceste comenzi actualizează cache-ul automat. Totuși, executarea lui după un `DROP USER` al utilizatorilor anonimi este o bună practică pentru a garanta consistența.
