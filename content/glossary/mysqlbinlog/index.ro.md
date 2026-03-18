---
title: "mysqlbinlog"
description: "Utilitar de linie de comandă MySQL pentru citirea, filtrarea și reaplicarea conținutului fișierelor binary log."
translationKey: "glossary_mysqlbinlog"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**mysqlbinlog** este utilitarul de linie de comandă furnizat cu MySQL pentru citirea și decodificarea conținutului fișierelor binary log. Este singurul instrument capabil să convertească formatul binar al binlog-urilor în output lizibil sau în instrucțiuni SQL re-executabile.

## Cum funcționează

mysqlbinlog citește fișierele binlog și produce output în format text. Suportă mai multe filtre:

- **Pe interval de timp**: `--start-datetime` și `--stop-datetime` pentru a limita output-ul la o fereastră temporală
- **Pe bază de date**: `--database` pentru a filtra evenimentele unei baze de date specifice
- **Pe poziție**: `--start-position` și `--stop-position` pentru a selecta evenimente specifice

Cu format ROW, flag-ul `--verbose` decodifică modificările la nivel de rând în format pseudo-SQL comentat, altfel output-ul este un blob binar ilizibil.

## La ce folosește

mysqlbinlog este utilizat în două scenarii principale:

- **Point-in-time recovery**: extragerea și reaplicarea evenimentelor de la backup până la momentul dorit, trimițând output-ul direct în clientul mysql
- **Debug de replicare**: analizarea evenimentelor pentru a înțelege ce a fost replicat, identificarea tranzacțiilor problematice sau reconstruirea secvenței de operațiuni care a cauzat o problemă

## Când se folosește

mysqlbinlog este esențial ori de câte ori trebuie să inspectezi ce s-a întâmplat în baza de date după un incident, sau când se execută un point-in-time recovery. Necesită acces la fișierele binlog pe filesystem-ul serverului sau posibilitatea de a se conecta la server cu `--read-from-remote-server`.
