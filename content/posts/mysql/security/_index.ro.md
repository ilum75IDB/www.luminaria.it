---
title: "Access & Control"
description: "Securitate MySQL și MariaDB în practică: modelul utilizator@host, privilegii granulare și capcanele unui sistem de autentificare care leagă identitatea de originea conexiunii."
layout: "list"
image: "security.cover.jpg"
---
În MySQL securitatea începe înainte de parolă.<br>
Începe cu întrebarea: de unde te conectezi?<br>

Pentru că MySQL nu te întreabă doar cine ești.<br>
Te întreabă cine ești **și de unde vii**.<br>
Iar răspunsul schimbă totul: privilegii, acces, chiar și existența ta ca utilizator.<br>

Este un model pe care alte baze de date nu îl au.<br>
Este puternic când îl stăpânești.<br>
Este o capcană când îl ignori.<br>

Aici vei găsi analize asupra modelului de autentificare MySQL și MariaDB, greșelile cele mai frecvente în gestionarea accesului și diferențele operaționale dintre cele două motoare pe care oricine lucrează în producție trebuie să le cunoască.<br>

Pentru că majoritatea problemelor de securitate pe MySQL nu provin din atacuri externe.<br>
Provin din utilizatori creați fără a înțelege cum îi interpretează motorul.
