---
title: "Least Privilege"
description: "Principiu de securitate care prevede atribuirea fiecărui utilizator sau proces doar a permisiunilor strict necesare pentru îndeplinirea funcției sale."
translationKey: "glossary_least-privilege"
aka: "Principiul privilegiului minim"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

**Least Privilege** (principiul privilegiului minim) este un principiu fundamental al securității informatice: fiecare utilizator, proces sau sistem trebuie să aibă doar permisiunile strict necesare pentru a-și îndeplini funcția, nimic mai mult.

## Cum funcționează

În contextul bazelor de date, principiul se aplică atribuind privilegii granulare: `SELECT` dacă utilizatorul trebuie doar să citească, `SELECT + INSERT + UPDATE` dacă trebuie și să scrie, niciodată `ALL PRIVILEGES` dacă nu este strict necesar. Combinat cu modelul `utilizator@host` al MySQL, principiul poate fi aplicat și în funcție de originea conexiunii.

## La ce folosește

Limitarea privilegiilor reduce suprafața de atac. Dacă o aplicație este compromisă, atacatorul moștenește privilegiile utilizatorului de bază de date al aplicației. Dacă acel utilizator are doar SELECT pe o bază de date specifică, daunele sunt limitate. Dacă are ALL PRIVILEGES, întregul server este în pericol.

## Când se folosește

Întotdeauna. Principiul least privilege este aplicabil în orice context: utilizatori de baze de date, utilizatori de sistem de operare, roluri aplicative, conturi de serviciu. Tentația de a atribui privilegii largi "pentru a nu avea probleme" este cauza cea mai frecventă a incidentelor de securitate evitabile.
