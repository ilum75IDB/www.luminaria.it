---
title: "La tecnica del Sì-E: come ho evitato una discussione che stava per esplodere"
description: "La tecnica del Yes-And, nata nel teatro di improvvisazione, applicata alla gestione dei conflitti nei team IT. Un caso reale di una riunione che stava degenerando — e come tre parole hanno cambiato tutto."
date: "2026-01-13T10:00:00+01:00"
draft: false
translationKey: "tecnica_si_e_yes_and"
tags: ["conflict-management", "team-leadership", "communication", "meeting"]
categories: ["Project Management"]
image: "tecnica-si-e-yes-and.cover.jpg"
---

Era un giovedì pomeriggio, una di quelle riunioni che sulla carta doveva durare un'ora. Eravamo in sette, collegati in call. L'ordine del giorno era semplice: decidere la strategia di migrazione di un database Oracle da on-premise a cloud.

Semplice, appunto. Sulla carta.

Dopo venti minuti, la riunione si era trasformata in un duello.

------------------------------------------------------------------------

## 🔥 La scintilla

Da una parte c'era il responsabile infrastruttura. Uomo di esperienza, vent'anni di datacenter alle spalle. La sua posizione era granitica: **migrazione lift-and-shift, zero modifiche all'architettura, si porta tutto così com'è**.

Dall'altra, il lead developer. Giovane, brillante, con le idee chiare. Voleva riscrivere lo strato applicativo, adottare servizi cloud-native, containerizzare tutto. **Rifacciamo da zero, tanto il codice è vecchio.**

Due posizioni legittime. Due prospettive reali. Due persone intelligenti.

Ma la conversazione aveva preso una piega familiare — e pericolosa.

"No, non ha senso portare tutto su cloud senza ripensare l'architettura."\
"No, riscrivere tutto è un rischio enorme e non abbiamo il budget."

No. No. No.

Ogni frase iniziava con "no". Ogni risposta era una negazione di quella precedente. Le braccia incrociate, il tono che saliva, le frasi che si accorciavano. Conosco quello schema. L'ho visto centinaia di volte. E so come finisce: non finisce. La riunione si chiude senza decisione, si rimanda alla prossima settimana, e nel frattempo nessuno fa niente perché "non abbiamo ancora deciso".

Il progetto si ferma. Non per motivi tecnici. Per orgoglio.

------------------------------------------------------------------------

## 🎭 Tre parole che cambiano tutto

A quel punto ho fatto una cosa molto semplice. Ho aspettato una pausa — perché nelle discussioni accese c'è sempre un momento in cui tutti riprendono fiato — e ho detto:

> "Marco, hai ragione: portare tutto su cloud senza cambiare nulla è il modo più veloce per andare in produzione. **E** potremmo anche identificare due o tre componenti che, migrando, ha senso ripensare in chiave cloud-native. Luca, tu quali sceglieresti?"

Nessuno ha detto "no". Nessuno è stato contraddetto.

Marco si è visto dare ragione — il suo approccio conservativo era il punto di partenza. Luca si è visto offrire un ruolo concreto — scegliere cosa modernizzare, con un mandato preciso.

In trenta secondi, due persone che stavano litigando si sono ritrovate a collaborare sulla stessa lavagna.

La riunione è finita in anticipo. Con una decisione. Una vera.

------------------------------------------------------------------------

## 🧠 Cos'è la tecnica del "Sì-E"

Quello che ho fatto ha un nome. Si chiama **"Yes-And"** — in italiano, **"Sì-E"**. Viene dal teatro di improvvisazione, dove c'è una regola fondamentale: **non negare mai la proposta del tuo partner di scena**.

Se qualcuno dice "Siamo su una barca in mezzo all'oceano", tu non rispondi "No, siamo in un ufficio". Rispondi "Sì, e sembra che stia arrivando una tempesta". Costruisci. Aggiungi. Vai avanti.

Nel project management funziona allo stesso modo.

Quando qualcuno propone qualcosa e tu rispondi "No, però...", succede questo a livello psicologico:

- l'interlocutore si mette sulla difensiva
- smette di ascoltare quello che viene dopo il "però"
- si concentra su come ribattere, non su come risolvere
- la conversazione diventa un ping-pong di negazioni

Quando rispondi "Sì, e...", succede il contrario:

- l'interlocutore si sente riconosciuto
- abbassa le difese
- diventa disponibile ad ascoltare la tua aggiunta
- la conversazione diventa costruttiva

Non è manipolazione. Non è diplomazia vuota. È una tecnica precisa per far progredire le decisioni senza bruciare i rapporti.

------------------------------------------------------------------------

## 🛠️ Come funziona nella pratica quotidiana

In trent'anni di progetti, ho applicato il "Sì-E" in decine di situazioni. Funziona ovunque ci sia una decisione da prendere e più persone con opinioni diverse.

### Nelle riunioni di progetto

**Invece di:** "No, la timeline di tre mesi è irrealistica."\
**Prova con:** "Sì, tre mesi è l'obiettivo. E per arrivarci dovremmo tagliare lo scope del primo rilascio a queste tre funzionalità — le altre le mettiamo nella fase due."

Noti la differenza? Nella prima versione hai un muro. Nella seconda hai un piano.

### Nelle code review

**Invece di:** "No, questo approccio è sbagliato, l'hai scritto in modo troppo complicato."\
**Prova con:** "Sì, funziona. E potremmo semplificarlo estraendo questa logica in un metodo separato — diventa più testabile."

Lo sviluppatore non si sente attaccato. Si sente aiutato. E la prossima volta viene da te a chiedere un parere *prima* di scrivere il codice, non dopo.

### Nelle negoziazioni con gli stakeholder

**Invece di:** "No, non possiamo aggiungere quella feature adesso, siamo già in ritardo."\
**Prova con:** "Sì, quella feature ha senso. E per inserirla senza compromettere la data di rilascio, dovremmo sostituirla con questa altra che è meno prioritaria. Quale delle due preferite?"

Lo stakeholder non sente un "no". Sente un "sì, e ora decidiamo insieme come farlo".

------------------------------------------------------------------------

## ⚠️ Quando il "Sì-E" non funziona

Sarebbe bello dire che funziona sempre. Non è così. Ci sono situazioni in cui il "Sì-E" è lo strumento sbagliato.

**Problemi di sicurezza.** Se qualcuno propone di togliere l'autenticazione dal database di produzione perché "rallenta le query", la risposta non è "Sì, e...". La risposta è "No. Punto."

**Violazioni di processo.** Se uno sviluppatore vuole fare il deploy in produzione il venerdì sera senza test, non c'è "Sì-E" che tenga. C'è un processo, e va rispettato.

**Deadline non negoziabili.** Quando il go-live è lunedì e siamo a giovedì, non è il momento di costruire sopra le idee di tutti. È il momento di decidere, eseguire e chiudere.

**Comportamenti tossici.** Il "Sì-E" funziona con persone in buona fede che hanno opinioni diverse. Non funziona con chi vuole solo avere ragione, con chi boicotta, con chi non ascolta per principio. In quei casi serve un altro tipo di conversazione — privata, diretta e molto franca.

La tecnica non è una formula magica. È uno strumento. E come tutti gli strumenti, devi sapere quando usarlo e quando metterlo giù.

------------------------------------------------------------------------

## 📊 Il costo nascosto del "No, però..."

Ho provato a fare un calcolo approssimativo su un progetto che ho gestito due anni fa. Un team di otto persone, riunioni tre volte a settimana.

| Situazione | Durata media riunione | Decisioni prese |
|---|---|---|
| Prima (cultura del "No, però...") | 1h 20min | 0.5 per riunione |
| Dopo (cultura del "Sì, e...") | 45min | 1.8 per riunione |

Il team prendeva decisioni tre volte più velocemente e le riunioni duravano quasi la metà.

Non ho numeri scientifici. Sono dati empirici, raccolti su un progetto specifico. Ma il pattern è coerente con quello che ho visto in vent'anni: i team che discutono in modo costruttivo vanno più veloci di quelli che litigano. Non perché evitano il conflitto — perché lo attraversano meglio.

------------------------------------------------------------------------

## 🎯 Quello che ho imparato

Il "Sì-E" non è diplomazia. Non è evitare il confronto. Non è dire sì a tutto.

È riconoscere che **la maggior parte delle discussioni nei progetti IT non riguarda chi ha ragione**. Riguarda come far progredire le cose. E le cose progrediscono quando le persone si sentono ascoltate, non quando vengono sconfitte.

Ho visto progetti bloccarsi per settimane perché due persone brillanti non riuscivano a smettere di dirsi "no" a vicenda. E ho visto gli stessi progetti sbloccarsi in quindici minuti quando qualcuno ha avuto il buon senso di dire "sì, e...".

Non serve un corso di comunicazione. Non serve un coach. Serve provare, la prossima volta che qualcuno dice qualcosa con cui non sei d'accordo, a rispondere "Sì, e..." invece di "No, però...".

Un esercizio semplice. Che cambia il modo in cui le decisioni vengono prese.\
Che cambia il modo in cui le persone lavorano insieme.\
E che, a volte, salva una riunione che stava per esplodere.

------------------------------------------------------------------------

## 💬 A chi è capitato almeno una volta

Se hai mai partecipato a una riunione dove due persone si parlavano addosso e nessuno ascoltava nessuno. Se hai mai visto un progetto bloccarsi non per un problema tecnico, ma per un problema di comunicazione. Se hai mai pensato "ma perché non riusciamo semplicemente a decidere?"

Prova il "Sì-E". La prossima riunione. Una volta sola.

Non costa niente. Non richiede approvazione. Non serve un budget.\
Serve solo la capacità di trattenerti un secondo prima di dire "no" — e sostituirlo con "sì, e...".

Il risultato potrebbe sorprenderti.

------------------------------------------------------------------------

## Glossario

**[Yes-And](/it/glossary/yes-and/)** — Tecnica di comunicazione nata nel teatro di improvvisazione che sostituisce il "No, però..." con "Sì, e...", trasformando le discussioni in costruzione collaborativa.

**[Stakeholder](/it/glossary/stakeholder/)** — Persona o gruppo con un interesse diretto nel risultato di un progetto: committente, utente finale, sponsor, team tecnico o qualsiasi parte influenzata dalle decisioni progettuali.

**[Scope](/it/glossary/scope/)** — Perimetro di un progetto che definisce cosa è incluso e cosa è escluso: funzionalità, deliverable, vincoli e confini concordati con gli stakeholder.

**[Lift-and-Shift](/it/glossary/lift-and-shift/)** — Strategia di migrazione che sposta un sistema da un ambiente a un altro senza modificarne l'architettura, il codice o la configurazione.

**[Timeboxing](/it/glossary/timeboxing/)** — Tecnica di gestione del tempo che assegna un intervallo fisso e non negoziabile a un'attività, forzando la conclusione entro il limite stabilito.
