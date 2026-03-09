---
title: "Access & Control"
description: "Sicurezza Oracle in pratica: utenti, ruoli, privilegi e audit. Il modello di sicurezza più granulare del mercato — e il più facile da configurare male."
layout: "list"
image: "security.cover.jpg"
---
In Oracle la sicurezza non è un'opzione.<br>
È una struttura portante del database, presente dal primo giorno.<br>

Il modello di sicurezza Oracle è probabilmente il più completo tra i database relazionali: utenti separati dagli schemi (ma collegati), ruoli predefiniti e custom, privilegi di sistema e di oggetto, profili di risorse, audit unificato.<br>

Potente? Enormemente.<br>
Mal configurato nel 90% delle installazioni che ho visto? Anche.<br>

Il problema non è la mancanza di strumenti. È l'eccesso. Oracle offre talmente tante opzioni di sicurezza che la tentazione è semplificare con un `GRANT DBA` e passare al problema successivo. L'ho visto fare da sviluppatori, da sistemisti, persino da DBA con anni di esperienza.<br>

In questa sezione racconto come si progetta la sicurezza Oracle in ambienti reali: separazione dei ruoli, principio del privilegio minimo, audit e le trappole in cui cadono anche i professionisti esperti.<br>

Perché in Oracle la sicurezza non si aggiunge dopo.<br>
Si progetta prima. O si paga poi.
