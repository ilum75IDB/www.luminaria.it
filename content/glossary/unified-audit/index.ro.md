---
title: "Unified Audit"
description: "Sistem de audit centralizat introdus în Oracle 12c care unifică toate tipurile de audit într-o singură infrastructură, înlocuind vechiul audit tradițional."
translationKey: "glossary_unified-audit"
aka: "Oracle Unified Auditing"
articles:
  - "/posts/oracle/oracle-roles-privileges"
---

**Unified Audit** (Oracle Unified Auditing) este sistemul de audit centralizat introdus în Oracle Database 12c care înlocuiește mecanismele de audit tradiționale cu o singură infrastructură unificată. Toate evenimentele de audit converg în vizualizarea `UNIFIED_AUDIT_TRAIL`.

## Cum funcționează

Unified Audit se bazează pe **audit policies**: reguli declarative care specifică ce acțiuni să fie monitorizate (DDL, DML, autentificări, operațiuni administrative). Politicile se creează cu `CREATE AUDIT POLICY`, se activează cu `ALTER AUDIT POLICY ... ENABLE` și pot fi aplicate utilizatorilor specifici sau global. Înregistrările de audit sunt scrise într-o coadă internă și apoi persistate în tabela de sistem.

## La ce servește

Răspunde la întrebarea fundamentală a securității: „cine a făcut ce, când și de unde?" Permite urmărirea operațiunilor critice precum DROP TABLE, GRANT, REVOKE, accesul la date sensibile și tentativele de autentificare eșuate. Este esențial pentru conformitate (GDPR, SOX, PCI-DSS) și pentru investigațiile post-incident.

## De ce contează

Vechiul audit tradițional al Oracle fragmenta jurnalele între fișiere de sistem, tabela SYS.AUD$ și FGA_LOG$, făcând analiza complexă. Unified Audit centralizează totul într-un singur punct, cu performanțe mai bune și gestionare simplificată. Într-un mediu fără audit configurat, un incident de securitate devine imposibil de reconstruit.
