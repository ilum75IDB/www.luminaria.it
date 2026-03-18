---
title: "systemd"
description: "Sistema di init e gestore dei servizi su Linux, usato per gestire istanze multiple di MySQL/MariaDB sullo stesso server tramite unit file separati."
translationKey: "glossary_systemd"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**systemd** è il sistema di init e il gestore dei servizi predefinito nelle distribuzioni Linux moderne (CentOS/RHEL 7+, Ubuntu 16.04+, Debian 8+). In ambito database, è il meccanismo che avvia, ferma e monitora le istanze MySQL o MariaDB.

## Come funziona

Ogni servizio è definito da un file unit (es. `mysqld.service`) che specifica il comando di avvio, il file di configurazione, le dipendenze e il comportamento in caso di crash. In un setup multi-istanza, si creano unit file separati per ogni istanza (es. `mysqld-app2.service`, `mysqld-reporting.service`), ciascuno con il proprio `--defaults-file` che punta a un `my.cnf` diverso.

## A cosa serve

systemd permette di gestire le istanze MySQL come servizi indipendenti: avviarle, fermarle, riavviarle e monitorarle separatamente. Il comando `systemctl cat <servizio>` è fondamentale per risalire dal nome del servizio al file di configurazione dell'istanza, e da lì a porta, socket e datadir.

## Quando si usa

systemd è attivo automaticamente su qualsiasi server Linux moderno. In ambito DBA, si interagisce con esso tramite `systemctl start/stop/status/restart <servizio>`. In ambienti multi-istanza, `systemctl list-units --type=service | grep mysql` è il primo comando per identificare quante istanze sono attive su un server.
