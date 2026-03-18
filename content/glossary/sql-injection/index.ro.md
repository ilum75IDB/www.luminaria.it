---
title: "SQL Injection"
description: "Tehnică de atac care inserează cod SQL malițios în input-urile unei aplicații pentru a manipula interogările executate de baza de date, accesând potențial date neautorizate sau compromițând sistemul."
translationKey: "glossary_sql-injection"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**SQL Injection** este una dintre cele mai răspândite și periculoase vulnerabilități în aplicațiile web. Apare când input-urile furnizate de utilizator sunt inserate direct în interogările SQL fără validare sau parametrizare, permițând unui atacator să modifice logica interogării.

## Cum funcționează

Atacatorul inserează fragmente de cod SQL în câmpurile de input ale aplicației (formulare de autentificare, câmpuri de căutare, parametri URL). Dacă aplicația concatenează aceste input-uri direct în interogările SQL, codul malițios este executat de baza de date cu privilegiile utilizatorului aplicativ. În combinație cu privilegiul `FILE` al MySQL-ului și un `secure-file-priv` neconfigurat, atacatorul poate citi fișiere de sistem sau scrie fișiere arbitrare pe server.

## La ce folosește

Înțelegerea SQL injection este fundamentală pentru cine gestionează baze de date în producție, deoarece multe configurații de securitate (precum `secure-file-priv`, gestionarea privilegiilor și separarea utilizatorilor) există specific pentru a mitiga impactul acestui tip de atac.

## Când se folosește

Termenul descrie un atac de prevenit, nu o tehnică de utilizat. Contramăsurile principale sunt: interogări parametrizate (prepared statements), validarea input-urilor, principiul privilegiului minim pentru utilizatorii bazei de date și configurarea corectă a directivelor precum `secure-file-priv`.
