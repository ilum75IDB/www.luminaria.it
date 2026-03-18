---
title: "systemd"
description: "Sistem de inițializare și manager de servicii pe Linux, folosit pentru gestionarea instanțelor multiple MySQL/MariaDB pe același server prin unit file separate."
translationKey: "glossary_systemd"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**systemd** este sistemul de inițializare și managerul de servicii implicit pe distribuțiile Linux moderne (CentOS/RHEL 7+, Ubuntu 16.04+, Debian 8+). În contextul bazelor de date, este mecanismul care pornește, oprește și monitorizează instanțele MySQL sau MariaDB.

## Cum funcționează

Fiecare serviciu este definit de un fișier unit (ex. `mysqld.service`) care specifică comanda de pornire, fișierul de configurare, dependențele și comportamentul în caz de crash. Într-un setup multi-instanță, se creează fișiere unit separate pentru fiecare instanță (ex. `mysqld-app2.service`, `mysqld-reporting.service`), fiecare cu propriul `--defaults-file` care indică spre un `my.cnf` diferit.

## La ce folosește

systemd permite gestionarea instanțelor MySQL ca servicii independente: pornirea, oprirea, repornirea și monitorizarea lor separat. Comanda `systemctl cat <serviciu>` e esențială pentru a urma traseul de la numele serviciului la fișierul de configurare al instanței, și de acolo la port, socket și datadir.

## Când se folosește

systemd este activ automat pe orice server Linux modern. În munca de DBA, interacționezi cu el prin `systemctl start/stop/status/restart <serviciu>`. În mediile multi-instanță, `systemctl list-units --type=service | grep mysql` este prima comandă pentru a identifica câte instanțe sunt active pe un server.
