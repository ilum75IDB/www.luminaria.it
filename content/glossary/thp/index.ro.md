---
title: "THP"
description: "Transparent Huge Pages — funcție a kernelului Linux care promovează automat paginile normale la pagini mari, dar care cauzează latențe imprevizibile și trebuie dezactivată pentru Oracle."
translationKey: "glossary_thp"
aka: "Transparent Huge Pages"
articles:
  - "/posts/oracle/oracle-linux-kernel"
---

**THP** (Transparent Huge Pages) este o funcție a kernelului Linux care promovează automat paginile de memorie de la 4 KB la 2 MB în fundal, fără configurare explicită. Spre deosebire de Huge Pages statice, sunt gestionate de procesul kernel `khugepaged`.

## Cum funcționează

Când sunt active (implicit `always`), kernelul încearcă să compacteze paginile normale în pagini mari în fundal. Procesul `khugepaged` lucrează continuu pentru a găsi și uni grupuri de pagini contigue, cauzând micro-înghețări imprevizibile în timpul operațiunilor de compactare.

## De ce contează

Pentru Oracle sunt un dezastru. Oracle declară explicit în documentație: dezactivați THP. "Blocajele de câteva secunde" de care utilizatorii se plâng sunt adesea cauzate de `khugepaged`. Se dezactivează cu `echo never > /sys/kernel/mm/transparent_hugepage/enabled` și prin GRUB pentru persistență la repornire.

## Ce poate merge prost

Confuzia între Huge Pages (bune pentru Oracle, configurate static) și THP (dăunătoare pentru Oracle, active implicit) este una dintre cele mai comune greșeli. Un DBA care vede "Huge Pages" în documentație și nu dezactivează THP face lucrurile mai rău în loc de mai bine.
