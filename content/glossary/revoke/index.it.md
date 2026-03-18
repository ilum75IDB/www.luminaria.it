---
title: "REVOKE"
description: "Comando SQL per rimuovere privilegi o ruoli precedentemente assegnati a un utente o ruolo, complementare al comando GRANT."
translationKey: "glossary_revoke"
aka: "Revoca privilegi SQL"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

**REVOKE** è il comando SQL che rimuove privilegi o ruoli precedentemente assegnati con `GRANT`. È il complemento indispensabile del GRANT e lo strumento principale per restringere i permessi quando un modello di sicurezza viene ristrutturato.

## Come funziona

La sintassi segue lo stesso schema del GRANT: `REVOKE SELECT ON schema.tabella FROM utente` oppure `REVOKE ruolo FROM utente`. In Oracle, revocare un ruolo come `DBA` rimuove in un colpo solo tutti i system privileges inclusi in quel ruolo. Prima di eseguire un REVOKE critico, è fondamentale verificare che l'utente mantenga i privilegi necessari per le sue funzioni.

## Quando si usa

Il caso più comune è la ristrutturazione del modello di sicurezza: rimuovere ruoli eccessivi (come DBA da utenti applicativi) e sostituirli con ruoli custom calibrati. Si usa anche quando un utente cambia funzione, quando un servizio viene dismesso, o quando un audit rivela privilegi assegnati in eccesso.

## Cosa può andare storto

Un REVOKE mal pianificato può bloccare applicazioni in produzione. Se un'applicazione si connette con un utente che perde il privilegio `CREATE SESSION`, smette di funzionare istantaneamente. Per questo la revoca di privilegi critici va sempre preceduta da un'analisi delle dipendenze e da un piano di rollout graduale.
