---
title: "INTO OUTFILE"
description: "Clauză SQL MySQL care permite scrierea rezultatului unui SELECT direct într-un fișier pe filesystem-ul serverului."
translationKey: "glossary_into-outfile"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**INTO OUTFILE** este o clauză SQL MySQL care permite exportul rezultatului unei interogări direct într-un fișier pe filesystem-ul serverului de baze de date. Este metoda nativă pentru generarea de fișiere CSV, TSV sau cu separatori personalizați.

## Cum funcționează

Clauza se adaugă la sfârșitul unui `SELECT` și specifică calea fișierului de destinație. Parametrii `FIELDS TERMINATED BY`, `ENCLOSED BY` și `LINES TERMINATED BY` controlează formatul output-ului. Fișierul este creat de utilizatorul de sistem MySQL (nu de utilizatorul care execută query-ul), deci trebuie să fie într-un director cu permisiunile corecte.

## La ce folosește

`INTO OUTFILE` e util pentru exporturi masive de date din baza de date în fișiere text structurate. Este complementul lui `LOAD DATA INFILE`, care face operațiunea inversă (importă date din fișiere). Împreună formează mecanismul nativ MySQL pentru import/export în masă.

## Când se folosește

Utilizarea e guvernată de directiva `secure-file-priv`: fișierul de destinație trebuie să fie în directorul autorizat. Când `secure-file-priv` blochează calea dorită, alternativa e folosirea clientului mysql din shell cu `-B -e` și redirecționarea output-ului, care nu e supusă acelorași restricții.
