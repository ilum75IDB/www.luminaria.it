---
title: "THP"
description: "Transparent Huge Pages — funzione del kernel Linux che promuove automaticamente le pagine normali a pagine grandi, ma che causa latenze imprevedibili e deve essere disabilitata per Oracle."
translationKey: "glossary_thp"
aka: "Transparent Huge Pages"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

Le **THP** (Transparent Huge Pages) sono una funzione del kernel Linux che promuove automaticamente le pagine di memoria da 4 KB a 2 MB in background, senza configurazione esplicita. A differenza delle Huge Pages statiche, sono gestite dal processo kernel `khugepaged`.

## Come funziona

Quando attive (default `always`), il kernel tenta di compattare le pagine normali in pagine grandi in background. Il processo `khugepaged` lavora continuamente per trovare e unire gruppi di pagine contigue, causando micro-freeze imprevedibili durante le operazioni di compattazione.

## Perché è critico

Per Oracle sono un disastro. Oracle lo dice esplicitamente nella documentazione: disabilitare THP. I "blocchi di qualche secondo" che gli utenti lamentano sono spesso causati da `khugepaged`. Si disabilitano con `echo never > /sys/kernel/mm/transparent_hugepage/enabled` e via GRUB per la persistenza al reboot.

## Cosa può andare storto

La confusione tra Huge Pages (buone per Oracle, configurate staticamente) e THP (dannose per Oracle, attive di default) è uno degli errori più comuni. Un DBA che vede "Huge Pages" nella documentazione e non disabilita THP sta peggiorando le cose invece di migliorarle.
