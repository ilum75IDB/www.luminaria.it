---
title: "Unix Socket"
description: "Meccanismo di comunicazione inter-processo locale su sistemi Unix/Linux, usato da MySQL per connessioni più veloci rispetto a TCP quando client e server sono sullo stesso host."
translationKey: "glossary_unix-socket"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

Un **Unix Socket** (o socket di dominio Unix) è un endpoint di comunicazione che permette a due processi sullo stesso sistema operativo di scambiarsi dati senza passare attraverso lo stack di rete TCP/IP. In ambito MySQL, è il metodo di connessione predefinito quando ci si collega a `localhost`.

## Come funziona

Quando un client MySQL si connette specificando `-h localhost`, il client non usa TCP. Usa il file socket Unix (tipicamente `/var/run/mysqld/mysqld.sock`) per comunicare direttamente con il processo del server MySQL. Questa comunicazione avviene interamente nel kernel, senza overhead di rete, ed è più veloce di una connessione TCP anche sullo stesso host.

## A cosa serve

In ambienti multi-istanza, ogni istanza MySQL ha il proprio file socket (es. `mysqld.sock`, `mysqld-app2.sock`). Specificare il socket corretto con `--socket=/path/to/socket` è l'unico modo affidabile per connettersi all'istanza desiderata. Senza specificare il socket, il client usa quello di default — che quasi sempre punta all'istanza primaria.

## Quando si usa

Il socket Unix si usa per tutte le connessioni locali a MySQL. In ambienti con istanze multiple, è essenziale specificare esplicitamente il socket per ogni connessione. Per connessioni remote (da un altro host), si usa invece TCP con `-h <ip> -P <porta>`.
