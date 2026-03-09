---
title: "Quando il caos diventa metodo: AI e GitHub per gestire un progetto che nessuno voleva toccare"
description: "Un caso reale di project management: come ho proposto GitHub e l'intelligenza artificiale per trasformare un progetto software caotico in un flusso di lavoro ordinato, misurabile e più veloce."
date: "2026-02-03T10:00:00+01:00"
draft: false
translationKey: "ai_github_project_management"
tags: ["ai", "github", "workflow", "bug-fixing", "software-evolution"]
categories: ["Project Management"]
image: "ai-github-project-management.cover.jpg"
---

Mi chiama un cliente. Voce tesa, parole misurate.

> "Ivan, abbiamo un problema. Anzi, abbiamo **il** problema."

Conosco quel tono. È il tono di chi ha già provato a risolvere le cose internamente, ha fallito, e adesso cerca qualcuno che gli dica la verità senza girarci intorno.

Il problema è un software gestionale — non un sito web, non un'app — un sistema critico su cui girano processi aziendali importanti. Ha qualche anno di vita. È cresciuto in fretta, come succede sempre quando il business corre più veloce dell'architettura. E adesso si è accumulato tutto: bug aperti che nessuno chiude, richieste evolutive che nessuno pianifica, sviluppatori che lavorano su versioni diverse del codice senza sapere cosa fa l'altro.

Il classico scenario che, a parole, "funziona". Ma che dentro è un campo minato.

------------------------------------------------------------------------

## 🧠 Il primo incontro: capire cosa non funziona davvero

Quando entro in un progetto come questo, non guardo subito il codice.\
Guardo le persone. Guardo come comunicano. Guardo dove si perdono le informazioni.

Il team era composto da quattro sviluppatori bravi. Seri. Competenti.\
Ma lavoravano così:

-   il codice stava su una cartella condivisa in rete
-   le modifiche venivano comunicate via email o su un foglio Excel
-   i bug venivano segnalati a voce, in chat, via ticket — senza un criterio unico
-   nessuno sapeva con certezza quale fosse la versione "buona" del software

E sai cosa succede in queste situazioni?\
Succede che ognuno ha ragione dal suo punto di vista. Ma il progetto, nel suo insieme, è fuori controllo.

Il problema non è tecnico. È organizzativo.\
E qui cambia tutto.

------------------------------------------------------------------------

## 📌 La proposta: GitHub come spina dorsale del progetto

La prima cosa che ho messo sul tavolo è stata chiara, diretta, senza fronzoli:

**Adottiamo GitHub. Tutto il codice passa da lì. Senza eccezioni.**

Non è una questione di moda. Non è perché "lo fanno tutti".\
È perché GitHub risolve, con strumenti concreti, problemi che nessun foglio Excel potrà mai gestire:

-   **Versioning reale**: ogni modifica è tracciata, commentata, reversibile
-   **Branch e Pull Request**: ogni sviluppatore lavora sulla sua copia, poi propone le modifiche al team — non sovrascrive il lavoro degli altri
-   **Issue tracker integrato**: i bug e le richieste evolutive vivono nello stesso posto del codice
-   **Cronologia completa**: chi ha fatto cosa, quando, perché

Ho visto la faccia dello sviluppatore senior. Un misto di curiosità e diffidenza.\
"Ma noi abbiamo sempre fatto così."

Gli ho risposto con calma: "Lo so. E il risultato è il motivo per cui sono qui."

Non l'ho detto per provocare. L'ho detto perché è la verità.\
E la verità, quando viene detta nel modo giusto, non offende. Libera.

------------------------------------------------------------------------

## 🔬 Il secondo passo: l'AI come acceleratore, non come sostituto

Una volta definito il flusso di lavoro su GitHub — branch, review, merge controllati — ho fatto la seconda proposta.

> "Integriamo l'intelligenza artificiale nel processo di risoluzione dei bug."

Silenzio.

Capisco la reazione. Quando dici "AI" in una stanza di sviluppatori, metà pensa a ChatGPT che genera codice a caso, l'altra metà pensa che gli stai dicendo che il loro lavoro non serve più.

Nessuna delle due cose.

Quello che ho proposto è molto diverso:

-   Quando uno sviluppatore prende in carico un bug, prima di scrivere una riga di codice, **usa l'AI per analizzare il contesto**
-   L'AI legge il codice coinvolto, i log, la descrizione del problema
-   Propone ipotesi. Non soluzioni definitive — **ipotesi ragionate**
-   Lo sviluppatore valuta, verifica, e poi implementa

L'AI non sostituisce il programmatore.\
L'AI gli fa risparmiare le prime due ore di analisi — quelle in cui stai leggendo codice scritto da qualcun altro, cercando di capire cosa diavolo succede.

E quelle due ore, moltiplicate per ogni bug, per ogni sviluppatore, per ogni settimana, diventano un numero che cambia i conti del progetto.

------------------------------------------------------------------------

## 📊 I numeri che ho messo sul tavolo

Non ho venduto sogni. Ho presentato stime conservative.

Il team gestiva mediamente 15-20 bug a settimana.\
Il tempo medio di risoluzione era di circa 6 ore per bug (tra analisi, fix, test, deploy).

Con l'introduzione di GitHub + AI nel workflow, la mia stima era:

| Metrica | Prima | Dopo (stima) |
|---|---|---|
| Tempo medio analisi bug | ~2.5 ore | ~15/20 minuti |
| Tempo totale risoluzione | ~6 ore | ~30 minuti |
| Bug risolti a settimana | 15-20 | 180-240 |
| Conflitti di codice | frequenti | rari |
| Visibilità stato progetto | nessuna | completa |

Una riduzione di oltre il 90% sul tempo totale di risoluzione.\
Un aumento di 12 volte sulla capacità del team di chiudere i ticket.\
Senza assumere nessuno. Senza cambiare le persone. Cambiando il metodo.

------------------------------------------------------------------------

## 🛠️ Come funziona in pratica

Il workflow che ho disegnato è semplice. Volutamente semplice.

**1. Il bug arriva come Issue su GitHub**\
Titolo chiaro, descrizione, label di priorità. Basta email, basta chat.

**2. Lo sviluppatore crea un branch dedicato**\
`fix/issue-234-errore-calcolo-iva` — il nome dice tutto.

**3. Prima di toccare il codice, interroga l'AI**\
Le passa il codice coinvolto, l'errore, il contesto. L'AI restituisce un'analisi strutturata: dove potrebbe essere il problema, quali file sono coinvolti, quali test verificare.

**4. Lo sviluppatore implementa il fix**\
Con un vantaggio enorme: sa già dove guardare.

**5. Pull Request con review**\
Un collega rivede il codice. Non per formalità — per qualità.

**6. Merge nel branch principale**\
Solo dopo l'approvazione. Il codice "buono" resta sempre buono.

**7. L'Issue si chiude automaticamente**\
Tracciabilità completa. Dal problema alla soluzione, tutto documentato.

------------------------------------------------------------------------

## 📈 Cosa è cambiato dopo tre settimane

I primi giorni sono stati i più duri. Sempre così.\
Nuovi strumenti, nuove abitudini, la tentazione di tornare al "come si faceva prima".

Ma dopo tre settimane è successo qualcosa.

Lo sviluppatore senior — quello che mi aveva guardato con diffidenza — mi ha scritto:

> "Ivan, ieri ho risolto un bug che l'anno scorso mi aveva bloccato per due giorni. Con l'AI ci ho messo quaranta minuti. Non perché l'AI ha scritto il codice. Ma perché mi ha fatto vedere subito dove stava il problema."

Ecco. Questo è il punto.\
L'AI non scrive codice migliore di uno sviluppatore esperto.\
L'AI **accelera il percorso** tra il problema e la comprensione del problema.

E la comprensione è sempre il passaggio più costoso.

------------------------------------------------------------------------

## 🎯 La lezione che porto a casa

Ogni volta che entro in un progetto in difficoltà, trovo lo stesso schema:

1.  Persone competenti
2.  Strumenti inadeguati
3.  Processi assenti o informali
4.  Frustrazione crescente

La soluzione non è mai "lavorare di più".\
La soluzione è **lavorare in modo diverso**.

GitHub non è un tool per sviluppatori. È un tool per **team**.\
L'AI non è un giocattolo. È un **moltiplicatore di competenza**.

Ma nessuno dei due funziona se non c'è qualcuno che guarda il progetto dall'alto, capisce dove si perdono le ore, e ha il coraggio di dire: "Cambiamo."

------------------------------------------------------------------------

## 💬 A chi si riconosce in questa storia

Se stai gestendo un progetto software e ti ritrovi in quello che ho descritto — il codice sparso, i bug che tornano, il team che lavora tanto ma chiude poco — sappi che non è colpa delle persone.

È colpa del sistema in cui lavorano.

E il sistema si può cambiare.\
Si **deve** cambiare.

Non servono rivoluzioni. Servono scelte precise, implementate con metodo.

Un repository condiviso. Un flusso di lavoro chiaro. Un assistente intelligente che accelera l'analisi.

Tre cose. Tre decisioni.\
Che trasformano il caos in controllo.
