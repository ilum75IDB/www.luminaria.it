---
title: "Tehnica Da-Și: cum am evitat o discuție care era pe punctul de a exploda"
description: "Tehnica Yes-And, născută în teatrul de improvizație, aplicată la gestionarea conflictelor în echipele IT. Un caz real al unei ședințe care degenera — și cum trei cuvinte au schimbat totul."
date: "2026-01-13T10:00:00+01:00"
draft: false
translationKey: "tecnica_si_e_yes_and"
tags: ["conflict-management", "team-leadership", "communication", "meeting"]
categories: ["Project Management"]
image: "tecnica-si-e-yes-and.cover.jpg"
---

Era o joi după-amiază, una dintre acele ședințe care pe hârtie trebuia să dureze o oră. Eram șapte, conectați într-un call. Ordinea de zi era simplă: să decidem strategia de migrare a unei baze de date Oracle din on-premise în cloud.

Simplă, desigur. Pe hârtie.

După douăzeci de minute, ședința se transformase într-un duel.

------------------------------------------------------------------------

## 🔥 Scânteia

De o parte era responsabilul de infrastructură. Om cu experiență, douăzeci de ani de datacentere în spate. Poziția lui era de granit: **migrare lift-and-shift, zero modificări la arhitectură, mutăm totul așa cum este**.

De cealaltă parte, lead developer-ul. Tânăr, strălucitor, cu idei clare. Voia să rescrie stratul aplicativ, să adopte servicii cloud-native, să containerizeze totul. **Refacem de la zero, oricum codul e vechi.**

Două poziții legitime. Două perspective reale. Doi oameni inteligenți.

Dar conversația luase o turnură familiară — și periculoasă.

"Nu, n-are sens să mutăm totul în cloud fără să regândim arhitectura."\
"Nu, să rescriem totul e un risc enorm și nu avem buget."

Nu. Nu. Nu.

Fiecare frază începea cu "nu". Fiecare răspuns era o negare a celui precedent. Brațele încrucișate, tonul ridicându-se, frazele scurtându-se. Cunosc acest tipar. L-am văzut de sute de ori. Și știu cum se termină: nu se termină. Ședința se închide fără o decizie, se reprogramează pentru săptămâna viitoare, și între timp nimeni nu face nimic pentru că "încă n-am decis".

Proiectul se blochează. Nu din motive tehnice. Din orgoliu.

------------------------------------------------------------------------

## 🎭 Trei cuvinte care schimbă totul

În acel moment am făcut ceva foarte simplu. Am așteptat o pauză — pentru că în discuțiile aprinse există întotdeauna un moment în care toți își trag sufletul — și am spus:

> "Marco, ai dreptate: să mutăm totul în cloud fără să schimbăm nimic e calea cea mai rapidă spre producție. **Și** am putea identifica și două-trei componente care, odată cu migrarea, are sens să fie regândite în cheie cloud-native. Luca, tu pe care le-ai alege?"

Nimeni n-a spus "nu". Nimeni n-a fost contrazis.

Marco și-a văzut poziția validată — abordarea lui conservatoare era punctul de plecare. Luca a primit un rol concret — să aleagă ce să modernizeze, cu un mandat precis.

În treizeci de secunde, doi oameni care se certau s-au regăsit colaborând pe aceeași tablă.

Ședința s-a terminat mai devreme. Cu o decizie. Una reală.

------------------------------------------------------------------------

## 🧠 Ce este tehnica "Da-Și"

Ceea ce am făcut are un nume. Se numește **"Yes-And"** — în română, **"Da-Și"**. Vine din teatrul de improvizație, unde există o regulă fundamentală: **nu nega niciodată propunerea partenerului tău de scenă**.

Dacă cineva spune "Suntem pe o barcă în mijlocul oceanului", nu răspunzi "Nu, suntem într-un birou". Răspunzi "Da, și pare că se apropie o furtună". Construiești. Adaugi. Mergi înainte.

În managementul de proiect funcționează la fel.

Când cineva propune ceva și răspunzi "Nu, dar...", iată ce se întâmplă la nivel psihologic:

- interlocutorul se pune pe poziție defensivă
- nu mai ascultă ce vine după "dar"
- se concentrează pe cum să contraargumenteze, nu pe cum să rezolve
- conversația devine un ping-pong de negații

Când răspunzi "Da, și...", se întâmplă contrariul:

- interlocutorul se simte recunoscut
- își coboară gardele
- devine disponibil să asculte adăugarea ta
- conversația devine constructivă

Nu e manipulare. Nu e diplomație goală. E o tehnică precisă pentru a face deciziile să avanseze fără a arde relațiile.

------------------------------------------------------------------------

## 🛠️ Cum funcționează în practica zilnică

În treizeci de ani de proiecte, am aplicat "Da-Și" în zeci de situații. Funcționează oriunde există o decizie de luat și mai multe persoane cu opinii diferite.

### În ședințele de proiect

**În loc de:** "Nu, timeline-ul de trei luni e nerealist."\
**Încearcă cu:** "Da, trei luni e obiectivul. Și ca să ajungem acolo, ar trebui să reducem scope-ul primului release la aceste trei funcționalități — restul le punem în faza doi."

Observi diferența? În prima versiune ai un zid. În a doua ai un plan.

### În code reviews

**În loc de:** "Nu, abordarea asta e greșită, ai scris prea complicat."\
**Încearcă cu:** "Da, funcționează. Și am putea simplifica extrăgând această logică într-o metodă separată — devine mai testabilă."

Developer-ul nu se simte atacat. Se simte ajutat. Și data viitoare vine la tine să ceară o părere *înainte* de a scrie codul, nu după.

### În negocierile cu stakeholderii

**În loc de:** "Nu, nu putem adăuga acea feature acum, suntem deja în întârziere."\
**Încearcă cu:** "Da, acea feature are sens. Și ca s-o includem fără să compromitem data de lansare, ar trebui s-o înlocuim cu aceasta care e mai puțin prioritară. Pe care dintre cele două o preferați?"

Stakeholderul nu aude un "nu". Aude un "da, și acum decidem împreună cum s-o facem".

------------------------------------------------------------------------

## ⚠️ Când "Da-Și" nu funcționează

Ar fi frumos să spun că funcționează mereu. Nu e așa. Sunt situații în care "Da-Și" e instrumentul greșit.

**Probleme de securitate.** Dacă cineva propune să scoată autentificarea de pe baza de date de producție pentru că "încetinește query-urile", răspunsul nu e "Da, și...". Răspunsul e "Nu. Punct."

**Încălcări de proces.** Dacă un developer vrea să facă deploy în producție vineri seara fără teste, nu există "Da-Și" care să țină. Există un proces, și trebuie respectat.

**Deadline-uri non-negociabile.** Când go-live-ul e luni și suntem joi, nu e momentul să construim pe ideile tuturor. E momentul să decidem, să executăm și să închidem.

**Comportamente toxice.** "Da-Și" funcționează cu oameni de bună-credință care au opinii diferite. Nu funcționează cu cei care vor doar să aibă dreptate, cu cei care sabotează, cu cei care nu ascultă din principiu. În acele cazuri e nevoie de alt tip de conversație — privată, directă și foarte sinceră.

Tehnica nu e o formulă magică. E un instrument. Și ca toate instrumentele, trebuie să știi când să-l folosești și când să-l lași deoparte.

------------------------------------------------------------------------

## 📊 Costul ascuns al lui "Nu, dar..."

Am încercat să fac un calcul aproximativ pe un proiect pe care l-am gestionat acum doi ani. O echipă de opt persoane, ședințe de trei ori pe săptămână.

| Situație | Durata medie a ședinței | Decizii luate |
|---|---|---|
| Înainte (cultura "Nu, dar...") | 1h 20min | 0.5 pe ședință |
| După (cultura "Da, și...") | 45min | 1.8 pe ședință |

Echipa lua decizii de trei ori mai repede, iar ședințele durau aproape jumătate.

Nu am date științifice. Sunt date empirice, colectate pe un proiect specific. Dar tiparul e consistent cu ce am văzut în douăzeci de ani: echipele care discută constructiv merg mai repede decât cele care se ceartă. Nu pentru că evită conflictul — pentru că-l traversează mai bine.

------------------------------------------------------------------------

## 🎯 Ce am învățat

"Da-Și" nu e diplomație. Nu e evitarea confruntării. Nu e să spui da la orice.

E să recunoști că **majoritatea discuțiilor din proiectele IT nu sunt despre cine are dreptate**. Sunt despre cum facem lucrurile să avanseze. Și lucrurile avansează când oamenii se simt ascultați, nu când sunt învinși.

Am văzut proiecte blocându-se săptămâni la rând pentru că două persoane strălucite nu reușeau să înceteze a-și spune "nu" reciproc. Și am văzut aceleași proiecte deblocându-se în cincisprezece minute când cineva a avut bunul simț să spună "da, și...".

Nu e nevoie de un curs de comunicare. Nu e nevoie de un coach. E nevoie să încerci, data viitoare când cineva spune ceva cu care nu ești de acord, să răspunzi "Da, și..." în loc de "Nu, dar...".

Un exercițiu simplu. Care schimbă modul în care se iau deciziile.\
Care schimbă modul în care oamenii lucrează împreună.\
Și care, uneori, salvează o ședință care era pe punctul de a exploda.

------------------------------------------------------------------------

## 💬 Pentru cine a pățit-o cel puțin o dată

Dacă ai fost vreodată într-o ședință unde doi oameni vorbeau peste ei și nimeni nu asculta pe nimeni. Dacă ai văzut vreodată un proiect blocându-se nu din cauza unei probleme tehnice, ci din cauza unei probleme de comunicare. Dacă te-ai gândit vreodată "de ce nu putem pur și simplu să decidem?"

Încearcă "Da-Și". La următoarea ședință. O singură dată.

Nu costă nimic. Nu are nevoie de aprobare. Nu necesită buget.\
Are nevoie doar de capacitatea de a te abține o secundă înainte de a spune "nu" — și de a-l înlocui cu "da, și...".

Rezultatul te-ar putea surprinde.

------------------------------------------------------------------------

## Glosar

**[Yes-And](/ro/glossary/yes-and/)** — Tehnică de comunicare născută în teatrul de improvizație care înlocuiește "Nu, dar..." cu "Da, și...", transformând discuțiile în construcție colaborativă.

**[Stakeholder](/ro/glossary/stakeholder/)** — Persoană sau grup cu un interes direct în rezultatul unui proiect: client, utilizator final, sponsor, echipă tehnică sau orice parte afectată de deciziile proiectului.

**[Scope](/ro/glossary/scope/)** — Perimetrul unui proiect care definește ce este inclus și ce este exclus: funcționalități, livrabile, constrângeri și limite convenite cu stakeholderii.

**[Lift-and-Shift](/ro/glossary/lift-and-shift/)** — Strategie de migrare care mută un sistem dintr-un mediu în altul fără a-i modifica arhitectura, codul sau configurația.

**[Timeboxing](/ro/glossary/timeboxing/)** — Tehnică de gestionare a timpului care atribuie un interval fix și nenegociabil unei activități, forțând încheierea în limita stabilită.

Rezultatul te-ar putea surprinde.
