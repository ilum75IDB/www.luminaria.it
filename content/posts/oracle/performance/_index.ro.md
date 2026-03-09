---
title: "Performance & Tuning"
date: "2026-03-10T10:00:00+01:00"
description: "Performanță Oracle în medii reale: partitioning, planuri de execuție, optimizarea interogărilor și strategii de tuning. Când datele cresc și interogările nu mai țin pasul."
layout: "list"
---
În Oracle performanța nu se măsoară după intuiție.<br>
Se măsoară cu numere: timp de răspuns, consistent gets, physical reads, wait events.<br>

Problema este că atunci când o bază de date este mică, totul funcționează. Interogările răspund în milisecunde, indecșii sunt suficienți, un full table scan nu face rău. Apoi datele cresc — milioane, sute de milioane, miliarde de rânduri — și ceea ce funcționa nu mai funcționează.<br>

Nu pentru că baza de date este stricată. Pentru că nu a fost proiectată pentru acea scală.<br>

În această secțiune împărtășesc cazuri reale de optimizare: tabele cu miliarde de rânduri, interogări care trec de la ore la secunde, planuri de execuție care ascund capcane invizibile. Nu teorie din manual, ci soluții aplicate în producție pe sisteme reale.<br>

Pentru că în Oracle tuning-ul nu este o artă mistică.<br>
Este inginerie. Și ca orice inginerie, se bazează pe măsurători, nu pe intuiții.
