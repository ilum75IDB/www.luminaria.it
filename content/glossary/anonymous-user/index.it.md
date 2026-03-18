---
title: "Anonymous User"
description: "Utente MySQL/MariaDB senza nome creato automaticamente durante l'installazione. Rappresenta un rischio di sicurezza perché può interferire con il matching degli utenti legittimi."
translationKey: "glossary_anonymous-user"
aka: "Utente anonimo"
articles:
  - "/posts/mysql/mysql-users-and-hosts"
---

L'**Anonymous User** (utente anonimo) è un account MySQL/MariaDB con username vuoto (`''@'localhost'`) che viene creato automaticamente durante l'installazione. Non ha nome e spesso non ha password.

## Come funziona

Quando un utente si connette, MySQL cerca la corrispondenza più specifica nella tabella `mysql.user`. L'utente anonimo `''@'localhost'` è più specifico di `'mario'@'%'` per una connessione da localhost, perché `'localhost'` batte `'%'` nella gerarchia di specificità. Di conseguenza, Mario che si connette da locale viene autenticato come utente anonimo e perde tutti i suoi privilegi.

## A cosa serve

L'utente anonimo era pensato per installazioni di sviluppo dove si voleva consentire connessioni senza credenziali. In produzione non ha alcuna utilità e rappresenta un rischio di sicurezza: può catturare connessioni destinate ad altri utenti e concedere accesso non autorizzato.

## Quando si usa

Mai in produzione. La prima operazione su qualsiasi installazione MySQL/MariaDB di produzione è verificare e rimuovere gli utenti anonimi con `SELECT user, host FROM mysql.user WHERE user = ''` seguito da `DROP USER ''@'localhost'`.
