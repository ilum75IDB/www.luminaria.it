---
title: "Disc plin pe un cluster MySQL: binary logs, Group Replication și o migrare care nu acceptă erori"
description: "Filesystem la 92% pe un cluster MySQL Group Replication cu 3 noduri. Cauza? Binary logs acumulate pe volumul principal. De la alertă la migrarea pe un volum dedicat, nod cu nod, fără a pierde quorum-ul."
date: "2025-10-14T08:03:00+01:00"
draft: false
translationKey: "mysql_group_replication_binlog_migration"
tags: ["group-replication", "binary-log", "disk-space", "cluster", "innodb-cluster"]
categories: ["mysql"]
image: "mysql-group-replication-binlog-migration.cover.jpg"
---

Alerta a venit într-o dimineață de luni, între trei ședințe și o cafea încă fierbinte. "Filesystem /mysql la 85% pe nodul primar." Pe un alt nod era la 66%, pe al treilea la 25%. Într-un cluster, când cifrele nu se potrivesc între noduri, întotdeauna e ceva dedesubt.

Prima întrebare care îți vine în minte este "cât spațiu mai trebuie?". Dar e întrebarea greșită. Cea corectă este: "de ce se umple?"

---

## Cauza: binary logs pe volumul greșit

Verificarea a fost rapidă:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

Rezultat: `ON`. Binary logs erau active — cum e de așteptat într-un cluster. Dar calea era problema:

```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```

```
/mysql/bin_log/binlog
```

Binlog-urile stăteau pe același volum cu datele: `/mysql`. Un volum de aproximativ 3 TB care pe un nod era deja la 85%.

Am verificat și retenția:

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

```
604800
```

Șapte zile. Nu e o valoare absurdă, dar cu trei noduri care scriu binlog-uri locale pe un volum partajat cu datele, șapte zile pot cântări mult — mai ales dacă încărcarea de scriere este mare.

Imaginea era clară: binary logs mâncau spațiul filesystem-ului principal. Nu un bug, nu o tabelă scăpată de sub control. Doar o alegere arhitecturală făcută la instalare și niciodată revizuită.

---

## Ce tip de cluster este, mai exact?

Înainte de a atinge orice pe un server MySQL — înainte chiar de a te gândi să muți un fișier — trebuie să știi ce ai în față. "E un cluster" nu e suficient. MySQL are cel puțin patru moduri diferite de a face high availability, și fiecare are regulile sale.

Am început cu replicarea clasică:

```sql
SHOW SLAVE STATUS\G
```

Empty set pe ambele noduri verificate. Nicio replicare tradițională activă.

Apoi am încercat `SHOW REPLICA STATUS` — dar pe MySQL 8.0.20 acea comandă nu există încă. A fost introdusă în 8.0.22. Un detaliu pe care documentația online adesea uită să-l menționeze, lăsându-te să urmărești o eroare de sintaxă care nu e una.

Pasul următor — Group Replication:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

Și acolo era răspunsul:

| MEMBER_HOST | MEMBER_STATE | MEMBER_ROLE |
|-------------|-------------|-------------|
| dbcluster01 | ONLINE | SECONDARY |
| dbcluster02 | ONLINE | SECONDARY |
| dbcluster03 | ONLINE | PRIMARY |

Trei noduri. Toate ONLINE. Un primary, două secondary. Group Replication în mod single-primary.

Confirmare finală din plugin-uri:

```sql
SHOW PLUGINS;
```

În listă: `group_replication | ACTIVE | GROUP REPLICATION | group_replication.so`. Și din configurație:

```sql
SHOW VARIABLES LIKE 'group_replication_single_primary_mode';
```

```
ON
```

Acum știam exact ce aveam în față. Nu replicare clasică, nu Galera, nu NDB Cluster. Un MySQL Group Replication single-primary cu trei noduri, GTID activat, format binlog ROW. Tabloul complet.

Tentația este întotdeauna să sari peste această fază. "Știu că e un cluster, hai să mergem." Dar a sări peste diagnostic pe un cluster e ca și cum ai opera fără un CT: poți avea noroc, sau poți provoca un dezastru.

---

## Soluția: un volum dedicat pentru binary logs

Strategia era simplă: binlog-urile au nevoie de propriul volum. Nu pe același filesystem cu datele, nu pe un symlink improvizat, nu pe un director partajat. Un volum dedicat, montat pe aceeași cale pe toate cele trei noduri.

Am cerut administratorilor de sistem să creeze un nou volum de 600 GB cu punct de montare `/mysql/binary_logs` pe fiecare dintre cele trei noduri.

Când volumul a fost gata, am verificat pe toate trei:

```bash
df -h /mysql/binary_logs
```

| Nod | /mysql | /mysql/binary_logs |
|-----|--------|--------------------|
| dbcluster03 (PRIMARY) | 85% | 1% |
| dbcluster02 (SECONDARY) | 66% | 1% |
| dbcluster01 (SECONDARY) | 25% | 1% |

Spațiu proaspăt și dedicat. Fiecare volum pe un disc local al VM-ului corespunzător — trei discuri, trei volume, același mountpoint pe toate cele trei noduri. Administratorii de sistem făcuseră o treabă curată.

---

## Verificările înainte de a atinge MySQL

Înainte de a opri primul nod, am rulat trei verificări pe care le consider obligatorii.

**Permisiunile directorului.** MySQL nu pornește dacă nu poate scrie în directorul de binlog. Pare evident, dar este una dintre cele mai frecvente cauze pentru "de ce nu repornește după schimbarea configurației?"

```bash
ls -ld /mysql/binary_logs
```

Pe toate cele trei noduri permisiunile erau 755. Funcționează, dar nu e ideal din punct de vedere al securității — binlog-urile pot conține date sensibile. Le-am schimbat la 750:

```bash
chmod 750 /mysql/binary_logs
```

Rezultat: `drwxr-x--- mysql mysql`. Doar utilizatorul mysql poate citi și scrie.

**Test de scriere real.** Înainte de a lăsa MySQL să scrie acolo, am verificat că filesystem-ul răspunde:

```bash
touch /mysql/binary_logs/testfile
ls -l /mysql/binary_logs/testfile
rm -f /mysql/binary_logs/testfile
```

Dacă touch eșuează, problema e de storage sau permisiuni — și mai bine să afli acum decât după un restart de MySQL.

**Starea clusterului.** Ultima verificare înainte de a continua:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

Trei noduri ONLINE. Quorum intact. Se poate porni.

---

## Strategia: un nod pe rând, primary-ul la urmă

Într-un Group Replication cu trei noduri, quorum-ul este doi. Dacă oprești un nod, celelalte două mențin grupul. Dacă oprești două — ai pierdut clusterul.

Regula e simplă: **un nod pe rând, așteptând ca precedentul să reintre în grup înainte de a atinge următorul**. Și primary-ul se face la urmă.

De ce? Pentru că atunci când oprești primary-ul, se întâmplă ceva important: clusterul declanșează o alegere automată și unul dintre secondary devine noul primary. În acele secunde — puține, dacă totul e sănătos — conexiunile active pot fi întrerupte, tranzacțiile în curs pot eșua. E o întrerupere scurtă, dar e o întrerupere. Trebuie comunicată.

Ordinea pe care am urmat-o:

1. **dbcluster01** (SECONDARY)
2. **dbcluster02** (SECONDARY)
3. **dbcluster03** (PRIMARY)

---

## Procedura, nod cu nod

Pe fiecare nod secvența este identică:

**A. Verifică rolul nodului.** Înainte de a-l opri, confirmă că e ceea ce crezi:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

**B. Oprește MySQL:**

```bash
systemctl stop mysqld
```

**C. Modifică configurația.** În `my.cnf`, schimbă parametrul `log_bin`:

Din:
```ini
log_bin=/mysql/bin_log/binlog
```

În:
```ini
log_bin=/mysql/binary_logs/mysql-bin
```

O linie. O singură modificare. Nu atinge parametrii Group Replication, nu schimba `server_id`, nu reinventa motorul cu aburi în timp ce schimbi o roată.

**D. Pornește MySQL:**

```bash
systemctl start mysqld
```

**E. Verifică.** Trei lucruri de controlat:

Noua cale:
```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```
Trebuie să returneze `/mysql/binary_logs/mysql-bin`.

Revenirea în grup:
```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```
Nodul trebuie să revină ONLINE.

Noile binlog-uri pe noua cale:
```bash
ls -lh /mysql/binary_logs/
```
Trebuie să apară noi fișiere `mysql-bin.000001`.

**Doar când nodul este ONLINE și clusterul arată din nou trei noduri active treci la următorul.** Nu înainte.

Pentru primary — dbcluster03 — procedura este identică, dar înainte de a-l opri am verificat că ambele secondary erau ONLINE și deja migrate. În momentul opririi, clusterul a declanșat alegerea. Unul dintre secondary a devenit primary. Scurtă întrerupere, cum era de așteptat.

---

## Ce să nu faci

Din experiența mea, acestea sunt cele mai comune capcane în acest tip de intervenție:

**Nu copia binlog-urile vechi pe noua cale.** În Group Replication nu e nevoie de arheologie binară. Noile binlog-uri se vor crea în noul director după restart. Cele vechi sunt necesare doar dacă ai nevoie de point-in-time recovery — și în cazul ăla știi deja unde le găsești.

**Nu atinge două noduri în același timp.** Cu trei noduri, quorum-ul e sacru. Un nod pe rând, fără excepții. Dacă oprești două împreună, joci Jenga cu ochii legați.

**Nu începe cu primary-ul.** Întotdeauna secondary întâi, primary la urmă. A face invers e modul elegant de a invita haosul la cină.

**Nu șterge binlog-urile vechi imediat.** După schimbare, calea veche `/mysql/bin_log/` nu va mai fi folosită pentru fișierele noi. Dar nu te grăbi cu `rm -rf /mysql/bin_log/*`. Așteaptă. Verifică că clusterul e stabil, că noile binlog-uri se scriu pe noul mount, că nu sunt erori în log-ul MySQL. Doar după câteva zile de observare, gândește-te la curățenie.

**Nu te baza doar pe faptul că "MySQL a pornit".** MySQL poate porni dar să nu reintre în grup. Trebuie să verifici trei lucruri: `log_bin_basename` indică noua cale, nodul este ONLINE în `replication_group_members`, și fișierele binlog se scriu efectiv în noul director.

---

## Ce învață această operație

Un filesystem la 92% nu e o urgență — e un semnal. Problema reală nu era spațiul pe disc, ci o alegere arhitecturală făcută la momentul instalării și niciodată revizuită: binlog-uri și date pe același volum.

Separarea binary logs pe un volum dedicat nu e doar un fix. E întărirea infrastructurii. E diferența dintre un sistem care "merge" și unul care e proiectat să meargă și când lucrurile cresc.

Și partea cea mai importantă a întregii intervenții nu a fost modificarea din `my.cnf` — aia e o linie. Partea importantă a fost diagnosticul: înțelegerea tipului de cluster, verificarea stării fiecărui nod, pregătirea storage-ului, testarea permisiunilor, planificarea ordinii de execuție. Totul înainte de a atinge un singur parametru.

Un DBA senior și un DBA junior cunosc amândoi comanda `systemctl stop mysqld`. Diferența e în tot ce se întâmplă înainte.

------------------------------------------------------------------------

## Glosar

**[Group Replication](/ro/glossary/group-replication/)** — Mecanismul nativ MySQL pentru replicare sincronă multi-nod cu failover automat și gestionarea quorum-ului. Suportă modurile single-primary și multi-primary.

**[Binary log](/ro/glossary/binary-log/)** — Registrul binar secvențial al MySQL care urmărește toate modificările de date (INSERT, UPDATE, DELETE, DDL), folosit pentru replicare și point-in-time recovery.

**[GTID](/ro/glossary/gtid/)** — Global Transaction Identifier — identificator unic atribuit fiecărei tranzacții în MySQL, care simplifică gestionarea replicării și urmărirea tranzacțiilor între nodurile clusterului.

**[Quorum](/ro/glossary/quorum/)** — Numărul minim de noduri care trebuie să fie active și în comunicare pentru ca un cluster să poată continua să opereze. Într-un cluster cu 3 noduri, quorum-ul este 2.

**[Single-primary](/ro/glossary/single-primary/)** — Modul Group Replication în care un singur nod acceptă scrieri, iar celelalte sunt read-only cu failover automat.
