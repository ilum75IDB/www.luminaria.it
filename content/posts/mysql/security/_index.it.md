---
title: "Access & Control"
description: "Sicurezza MySQL e MariaDB in pratica: il modello utente@host, i privilegi granulari e le trappole di un sistema di autenticazione che lega l'identità all'origine della connessione."
layout: "list"
image: "security.cover.jpg"
---
In MySQL la sicurezza inizia prima ancora della password.<br>
Inizia dalla domanda: da dove ti stai connettendo?<br>

Perché MySQL non ti chiede solo chi sei.<br>
Ti chiede chi sei **e da dove arrivi**.<br>
E la risposta cambia tutto: privilegi, accesso, persino la tua esistenza come utente.<br>

È un modello che altri database non hanno.<br>
È potente quando lo padroneggi.<br>
È una trappola quando lo ignori.<br>

Qui trovi analisi sul modello di autenticazione di MySQL e MariaDB, sugli errori più comuni nella gestione degli accessi e sulle differenze operative tra i due motori che chi lavora in produzione deve conoscere.<br>

Perché la maggior parte dei problemi di sicurezza su MySQL non nasce da attacchi esterni.<br>
Nasce da utenti creati senza capire come il motore li interpreta.
