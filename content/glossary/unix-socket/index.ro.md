---
title: "Unix Socket"
description: "Mecanism de comunicare inter-proces local pe sisteme Unix/Linux, folosit de MySQL pentru conexiuni mai rapide decât TCP când clientul și serverul sunt pe același host."
translationKey: "glossary_unix-socket"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

Un **Unix Socket** (sau socket de domeniu Unix) este un endpoint de comunicare care permite la două procese de pe același sistem de operare să facă schimb de date fără a trece prin stiva de rețea TCP/IP. În MySQL, este metoda de conectare implicită când te conectezi la `localhost`.

## Cum funcționează

Când un client MySQL se conectează specificând `-h localhost`, clientul nu folosește TCP. Folosește fișierul socket Unix (de obicei `/var/run/mysqld/mysqld.sock`) pentru a comunica direct cu procesul serverului MySQL. Această comunicare se desfășoară integral în kernel, fără overhead de rețea, și e mai rapidă decât o conexiune TCP chiar și pe același host.

## La ce folosește

În mediile multi-instanță, fiecare instanță MySQL are propriul fișier socket (ex. `mysqld.sock`, `mysqld-app2.sock`). Specificarea socket-ului corect cu `--socket=/path/to/socket` e singura metodă fiabilă de a te conecta la instanța dorită. Fără a specifica socket-ul, clientul îl folosește pe cel implicit — care aproape întotdeauna indică spre instanța primară.

## Când se folosește

Socket-urile Unix se folosesc pentru toate conexiunile locale la MySQL. În mediile cu instanțe multiple, e esențial să specifici explicit socket-ul pentru fiecare conexiune. Pentru conexiuni de la distanță (de pe alt host), se folosește TCP cu `-h <ip> -P <port>`.
