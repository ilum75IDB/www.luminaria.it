---
title: "Performance & Tuning"
date: "2026-03-10T10:00:00+01:00"
description: "Performance Oracle in ambienti reali: partitioning, execution plan, ottimizzazione query e strategie di tuning. Quando i dati crescono e le query non tengono il passo."
layout: "list"
---
In Oracle le performance non si misurano a sentimento.<br>
Si misurano con i numeri: tempo di risposta, consistent gets, physical reads, wait events.<br>

Il problema è che quando un database è piccolo, tutto funziona. Le query rispondono in millisecondi, gli indici bastano, il full table scan non fa paura. Poi i dati crescono — milioni, centinaia di milioni, miliardi di righe — e quello che funzionava smette di funzionare.<br>

Non perché il database sia rotto. Perché non è stato progettato per quella scala.<br>

In questa sezione racconto casi reali di ottimizzazione: tabelle da miliardi di righe, query che passano da ore a secondi, execution plan che nascondono trappole invisibili. Non teoria da manuale, ma soluzioni applicate in produzione su sistemi veri.<br>

Perché in Oracle il tuning non è un'arte mistica.<br>
È ingegneria. E come ogni ingegneria, si basa su misure, non su intuizioni.
