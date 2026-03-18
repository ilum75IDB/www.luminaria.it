---
title: "Unified Audit"
description: "Sistema di audit centralizzato introdotto in Oracle 12c che unifica tutti i tipi di audit in un'unica infrastruttura, sostituendo il vecchio audit tradizionale."
translationKey: "glossary_unified-audit"
aka: "Oracle Unified Auditing"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

**Unified Audit** (Oracle Unified Auditing) è il sistema di audit centralizzato introdotto in Oracle Database 12c che sostituisce i meccanismi di audit tradizionali con un'unica infrastruttura unificata. Tutti gli eventi di audit convergono nella vista `UNIFIED_AUDIT_TRAIL`.

## Come funziona

Unified Audit si basa su **audit policy**: regole dichiarative che specificano quali azioni monitorare (DDL, DML, login, operazioni amministrative). Le policy si creano con `CREATE AUDIT POLICY`, si attivano con `ALTER AUDIT POLICY ... ENABLE` e possono essere applicate a utenti specifici o globalmente. I record di audit vengono scritti in una coda interna e poi persistiti nella tabella di sistema.

## A cosa serve

Risponde alla domanda fondamentale della sicurezza: "chi ha fatto cosa, quando e da dove?" Permette di tracciare operazioni critiche come DROP TABLE, GRANT, REVOKE, accessi a dati sensibili e tentativi di login falliti. È essenziale per la compliance (GDPR, SOX, PCI-DSS) e per le indagini post-incidente.

## Perché è critico

Il vecchio audit tradizionale di Oracle frammentava i log tra file di sistema, tabelle SYS.AUD$ e FGA_LOG$, rendendo l'analisi complessa. Unified Audit centralizza tutto in un unico punto, con performance migliori e gestione semplificata. In un ambiente senza audit configurato, un incidente di sicurezza diventa impossibile da ricostruire.
