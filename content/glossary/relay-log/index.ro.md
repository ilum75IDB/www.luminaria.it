---
title: "Relay log"
description: "Fișier de log intermediar pe slave-ul MySQL care primește evenimentele din binary log-ul master-ului înainte de a fi executate local."
translationKey: "glossary_relay-log"
articles:
  - "/posts/mysql/binary-log-mysql"
---

**Relay log-ul** este un fișier de log intermediar prezent pe slave într-o arhitectură de replicare MySQL. Conține evenimentele primite din binary log-ul master-ului, în așteptarea de a fi executate local de către thread-ul SQL al slave-ului.

## Cum funcționează

Fluxul replicării MySQL trece prin relay log în trei faze:

1. **I/O thread-ul** slave-ului se conectează la master și citește binary log-urile
2. Evenimentele primite sunt scrise în relay log-ul local
3. **SQL thread-ul** slave-ului citește evenimentele din relay log și le execută pe baza de date locală

Această arhitectură cu două thread-uri permite decuplarea recepției datelor de aplicarea lor: slave-ul poate continua să primească evenimente de la master chiar dacă execuția locală este temporar mai lentă.

## La ce folosește

Relay log-ul este mecanismul care garantează consistența replicării. Funcționează ca un buffer între master și aplicarea locală a evenimentelor, permițând slave-ului să gestioneze diferențele de viteză fără a pierde date.

## Când se folosește

Relay log-ul este creat automat când se configurează replicarea MySQL. Nu necesită gestionare manuală directă, dar starea sa (poziția curentă, eventuala întârziere) este vizibilă prin `SHOW REPLICA STATUS` și este fundamentală pentru diagnosticarea problemelor de replica lag.
