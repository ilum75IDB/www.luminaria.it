---
title: "REVOKE"
description: "Comandă SQL pentru eliminarea privilegiilor sau rolurilor acordate anterior unui utilizator sau rol, complementară comenzii GRANT."
translationKey: "glossary_revoke"
aka: "Revocare privilegii SQL"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

**REVOKE** este comanda SQL care elimină privilegiile sau rolurile acordate anterior cu `GRANT`. Este complementul indispensabil al GRANT și instrumentul principal pentru restricționarea permisiunilor atunci când un model de securitate este restructurat.

## Cum funcționează

Sintaxa urmează același tipar ca GRANT: `REVOKE SELECT ON schema.tabela FROM utilizator` sau `REVOKE rol FROM utilizator`. În Oracle, revocarea unui rol precum `DBA` elimină dintr-o singură lovitură toate privilegiile de sistem incluse în acel rol. Înainte de a executa un REVOKE critic, este esențial să se verifice că utilizatorul păstrează privilegiile necesare pentru funcțiile sale.

## Când se folosește

Cel mai frecvent caz este restructurarea modelului de securitate: eliminarea rolurilor excesive (precum DBA de la utilizatorii aplicativi) și înlocuirea lor cu roluri personalizate calibrate. Se folosește și când un utilizator își schimbă funcția, când un serviciu este dezafectat, sau când un audit relevă privilegii acordate în exces.

## Ce poate merge prost

Un REVOKE prost planificat poate bloca aplicații în producție. Dacă o aplicație se conectează cu un utilizator care pierde privilegiul `CREATE SESSION`, încetează să funcționeze instantaneu. De aceea revocarea privilegiilor critice trebuie întotdeauna precedată de o analiză a dependențelor și un plan de implementare graduală.
