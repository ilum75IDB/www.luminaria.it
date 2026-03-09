---
title: "Galera Cluster cu 3 noduri: cum am rezolvat o problemă de disponibilitate pe MySQL"
description: "Un client cu un MySQL standalone care cădea în fiecare lună, luând cu el întreaga aplicație. Soluția mea: un Galera Cluster cu 3 noduri și replicare sincronă. De la diagnostic la punerea în producție, cu toate fișierele de configurare și parametrii critici."
date: "2026-02-24T10:00:00+01:00"
draft: false
translationKey: "galera_cluster_3_nodi"
tags: ["mysql", "mariadb", "galera", "cluster", "disponibilitate ridicată", "replicare", "wsrep", "linux"]
categories: ["mysql"]
image: "galera-cluster-3-nodi.cover.jpg"
---

Tichetul era laconic, cum se întâmplă adesea când problema e gravă: "Baza de date a căzut din nou. Aplicația e oprită. A treia oară în două luni."

Clientul avea un MariaDB pe un singur server Linux — o aplicație de gestiune de afaceri folosită de aproximativ două sute de utilizatori interni, cu vârfuri de încărcare în timpul închiderilor contabile de sfârșit de lună. De fiecare dată când serverul avea o problemă — un disc care se încetinea, o actualizare de sistem care necesita restart, un proces care consuma toată memoria RAM — baza de date cădea și cu ea întreaga operativitate a companiei.

Întrebarea nu era "cum reparăm serverul". Întrebarea era: **cum facem astfel încât data viitoare când un server are o problemă, aplicația să continue să funcționeze?**

Răspunsul, după douăzeci de ani de experiență cu acest tip de scenarii, era unul singur: **Galera Cluster**.

---

## Diagnosticul: un single point of failure clasic

Primul lucru pe care l-am făcut a fost să analizez infrastructura. Tabloul era familiar:

- Un singur server MariaDB, nicio replică
- Backup nocturn pe disc extern (măcar atât)
- Niciun mecanism de failover
- Aplicația indica direct IP-ul serverului de baze de date

Fiecare oprire, chiar și de zece minute, însemna două sute de persoane blocate. În timpul închiderilor contabile, însemna întârzieri care se propagau în cascadă în procesele de afaceri.

Am propus clientului o soluție bazată pe Galera Cluster: trei noduri MariaDB cu replicare sincronă multi-master. Orice nod acceptă citiri și scrieri, datele sunt coerente pe toate trei, iar dacă un nod cade, celelalte două continuă să servească aplicația fără întrerupere.

Clientul avea deja la dispoziție trei VM Linux — echipa de infrastructură le provizionase pentru un alt proiect care fusese amânat. Perfect: nu trebuia nici măcar să comande hardware.

---

## Planul: trei noduri, zero single point of failure

Mașinile disponibile:

| Nod | Hostname | IP |
|-----|----------|-----|
| Nod 1 | `db-node1` | `10.0.1.11` |
| Nod 2 | `db-node2` | `10.0.1.12` |
| Nod 3 | `db-node3` | `10.0.1.13` |

O premisă importantă: **Galera nu este o opțiune nativă în MySQL Community**. Se folosește MariaDB (care îl integrează nativ) sau Percona XtraDB Cluster (bazat pe MySQL). Clientul folosea deja MariaDB, deci alegerea era naturală și nu necesita migrare de motor.

Obiectivul era clar: migrarea de la o arhitectură cu un singur nod la un cluster cu trei noduri, fără a modifica aplicația în afară de adresa de conexiune.

---

## Instalare: aceeași versiune pe toate nodurile

Primul principiu nenegociabil: **toate nodurile trebuie să aibă exact aceeași versiune de MariaDB**. Am văzut clustere instabile luni de zile pentru că cineva actualizase un nod și nu pe celelalte.

Pe toate cele trei servere:

```bash
# Adăugarea repository-ului oficial MariaDB (exemplu pentru 11.4 LTS)
curl -LsS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | \
    sudo bash -s -- --mariadb-server-version="mariadb-11.4"

# Instalarea MariaDB Server și a plugin-ului Galera
sudo dnf install MariaDB-server MariaDB-client galera-4 -y

# Activare dar NU porni serviciul încă
sudo systemctl enable mariadb
```

Nu porni serviciul. Mai întâi se configurează. Întotdeauna.

---

## Inima configurării: `/etc/my.cnf.d/galera.cnf`

Acesta este fișierul care definește comportamentul clusterului. Trebuie creat pe fiecare nod, cu diferențele corespunzătoare pentru adresa IP și numele nodului.

Iată configurarea completă pentru **Nodul 1**:

```ini
[mysqld]
# === Motor și charset ===
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
innodb_flush_log_at_trx_commit=2
innodb_buffer_pool_size=1G

# === Configurare WSREP (Galera) ===
wsrep_on=ON
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so

# Lista TUTUROR nodurilor din cluster
wsrep_cluster_address="gcomm://10.0.1.11,10.0.1.12,10.0.1.13"

# Numele clusterului (trebuie să fie identic pe toate nodurile)
wsrep_cluster_name="galera_productie"

# Identitatea ACESTUI nod (se schimbă pe fiecare server)
wsrep_node_address="10.0.1.11"
wsrep_node_name="db-node1"

# Metoda SST (State Snapshot Transfer)
wsrep_sst_method=mariabackup
wsrep_sst_auth="sst_user:parola_sigura_aici"

# === Rețea ===
bind-address=0.0.0.0
```

Pentru **Nodul 2** și **Nodul 3**, singurul lucru care se schimbă este:

```ini
# Nod 2
wsrep_node_address="10.0.1.12"
wsrep_node_name="db-node2"

# Nod 3
wsrep_node_address="10.0.1.13"
wsrep_node_name="db-node3"
```

Tot restul este identic. **Identic**. Nu ceda tentației de a "personaliza" buffer pool sau alți parametri per nod: într-un cluster Galera, simetria este o virtute.

---

## De ce contează fiecare parametru

Să vedem parametrii unul câte unul, pentru că fiecare are un motiv precis.

### `binlog_format=ROW`

Galera necesită formatul ROW pentru binary log. Nu STATEMENT, nu MIXED. **ROW** și atât. Cu alte formate clusterul nici nu pornește — și pe bună dreptate, deoarece replicarea sincronă bazată pe certification depinde de precizia la nivel de rând.

### `innodb_autoinc_lock_mode=2`

Acest parametru setează modul de blocare pentru auto-increment la "interleaved". Într-un cluster multi-master, două noduri pot genera INSERT simultan pe aceeași tabelă. Cu lock mode 1 (implicit) s-ar crea deadlock-uri. Cu valoarea 2, InnoDB generează auto-increment-urile fără lock global, permițând inserări concurente de pe noduri diferite.

Consecința: ID-urile auto-increment **nu vor fi secvențiale** între noduri. Dacă aplicația ta depinde de secvențialitatea ID-urilor, ai o problemă arhitecturală de rezolvat în amonte.

### `innodb_flush_log_at_trx_commit=2`

Aici se face un compromis conștient. Valoarea 1 (implicită) garantează durabilitate totală — fiecare commit este scris și fsynced pe disc. Dar într-un cluster Galera, durabilitatea este deja garantată de replicarea sincronă pe trei noduri. Valoarea 2 scrie în buffer-ul sistemului de operare la fiecare commit și face fsync doar în fiecare secundă, îmbunătățind performanța de scriere cu 30-40% în testele noastre.

Dacă pierzi un nod, datele sunt pe celelalte două. Dacă pierzi întregul datacenter... ei bine, asta e altă discuție.

### `wsrep_sst_method=mariabackup`

SST este mecanismul prin care un nod care se alătură clusterului primește o copie completă a datelor. Opțiunile sunt:

| Metodă | Pro | Contra |
|--------|-----|--------|
| `rsync` | Rapid | Nodul donator se blochează pe citire |
| `mariabackup` | Nu blochează donatorul | Necesită instalare separată |
| `mysqldump` | Simplu | Foarte lent, blochează donatorul |

**Întotdeauna mariabackup**. În producție, blocarea unui nod donator în timpul unui SST înseamnă degradarea clusterului exact în momentul în care ai cea mai mare nevoie de el.

```bash
# Instalarea mariabackup pe toate nodurile
sudo dnf install MariaDB-backup -y
```

---

## Firewall: porturile pe care Galera le necesită deschise

Acesta este punctul în care văd 50% din primele instalări eșuând. Galera nu folosește doar portul MySQL.

```bash
# Pe toate cele trei noduri
sudo firewall-cmd --permanent --add-port=3306/tcp   # MySQL standard
sudo firewall-cmd --permanent --add-port=4567/tcp   # Comunicare cluster Galera
sudo firewall-cmd --permanent --add-port=4567/udp   # Replicare multicast (opțional)
sudo firewall-cmd --permanent --add-port=4568/tcp   # IST (Incremental State Transfer)
sudo firewall-cmd --permanent --add-port=4444/tcp   # SST (State Snapshot Transfer)
sudo firewall-cmd --reload
```

Dacă SELinux este activ (și ar trebui să fie), ai nevoie și de politici:

```bash
sudo setsebool -P mysql_connect_any 1
```

Patru porturi. Patru. Nici unul în plus, nici unul în minus. Dacă uiți unul, clusterul se formează dar nu sincronizează — iar debug-ul devine un exercițiu de frustrare.

---

## Migrarea datelor și bootstrap-ul

Înainte de a porni clusterul, am migrat datele de pe serverul standalone pe Nodul 1 cu un dump complet:

```bash
# Pe vechiul server standalone
mysqldump --all-databases --single-transaction --routines --triggers \
    --events > /tmp/full_dump.sql

# Transfer pe Nodul 1
scp /tmp/full_dump.sql db-node1:/tmp/
```

Apoi bootstrap-ul — momentul adevărului. Primul nod nu se pornește cu `systemctl start mariadb`. Se pornește cu comanda de bootstrap.

**Doar pe Nodul 1:**

```bash
sudo galera_new_cluster
```

Această comandă pornește MariaDB cu `wsrep_cluster_address=gcomm://` (gol), ceea ce înseamnă: "Eu sunt fondatorul, nu caut alte noduri."

Importul datelor și crearea utilizatorului SST:

```sql
-- Importul dump-ului de pe vechiul server
SOURCE /tmp/full_dump.sql;

-- Crearea utilizatorului pentru transferul de date între noduri
CREATE USER 'sst_user'@'localhost' IDENTIFIED BY 'parola_sigura_aici';
GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sst_user'@'localhost';
FLUSH PRIVILEGES;
```

Verificare imediată:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | wsrep_cluster_size | 1     |
-- +--------------------+-------+
```

Dacă valoarea este 1, bootstrap-ul a funcționat. Acum, **pe celelalte două noduri:**

```bash
sudo systemctl start mariadb
```

Aceste noduri citesc `wsrep_cluster_address`, găsesc Nodul 1, primesc un SST complet cu toate datele și se alătură clusterului.

După pornirea tuturor celor trei:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | wsrep_cluster_size | 3     |
-- +--------------------+-------+
```

Trei. Acesta este numărul magic. Momentul în care clientul a încetat să mai aibă un single point of failure.

---

## Verificarea stării de sănătate a clusterului

Aceasta este partea care contează cu adevărat pentru cine gestionează clusterul zi de zi. Nu este suficient să știi că `wsrep_cluster_size` este 3. Trebuie să știi să citești starea completă.

### Query-ul de diagnostic pe care îl folosesc mereu

```sql
SHOW STATUS WHERE Variable_name IN (
    'wsrep_cluster_size',
    'wsrep_cluster_status',
    'wsrep_connected',
    'wsrep_ready',
    'wsrep_local_state_comment',
    'wsrep_incoming_addresses',
    'wsrep_evs_state',
    'wsrep_flow_control_paused',
    'wsrep_local_recv_queue_avg',
    'wsrep_local_send_queue_avg',
    'wsrep_cert_deps_distance'
);
```

### Cum să interpretezi rezultatele

| Variabilă | Valoare sănătoasă | Semnificație |
|-----------|-------------------|--------------|
| `wsrep_cluster_size` | `3` | Toate nodurile sunt în cluster |
| `wsrep_cluster_status` | `Primary` | Clusterul este operațional și are quorum |
| `wsrep_connected` | `ON` | Acest nod este conectat la cluster |
| `wsrep_ready` | `ON` | Acest nod acceptă interogări |
| `wsrep_local_state_comment` | `Synced` | Acest nod este sincronizat |
| `wsrep_flow_control_paused` | `0.0` | Fără pauze de flow control |
| `wsrep_local_recv_queue_avg` | `< 0.5` | Coada de recepție este sub control |
| `wsrep_local_send_queue_avg` | `< 0.5` | Coada de trimitere este sub control |

### Semnalele de pericol

**`wsrep_cluster_status = Non-Primary`**: nodul a pierdut quorum-ul. Este izolat. Nu va accepta scrieri (și nu trebuie). Acest lucru se întâmplă când un nod pierde conexiunea cu majoritatea clusterului.

**`wsrep_flow_control_paused > 0.0`**: flow control activat. Înseamnă că un nod este prea lent în aplicarea tranzacțiilor și le cere celorlalte să încetinească. O valoare apropiată de 1.0 înseamnă că clusterul este în esență oprit, așteptând nodul cel mai lent.

**`wsrep_local_recv_queue_avg > 1.0`**: tranzacțiile care sosesc se acumulează. Ar putea fi o problemă de I/O disc, CPU, sau un nod subdimensionat.

### Script de monitorizare

Am livrat clientului și un script pentru sistemul lor de monitorizare (Zabbix, în cazul lor):

```bash
#!/bin/bash
# galera_health_check.sh — de executat pe fiecare nod

MYSQL="mysql -u monitor -p'monitor_pwd' -Bse"

CLUSTER_SIZE=$($MYSQL "SHOW STATUS LIKE 'wsrep_cluster_size'" | awk '{print $2}')
CLUSTER_STATUS=$($MYSQL "SHOW STATUS LIKE 'wsrep_cluster_status'" | awk '{print $2}')
NODE_STATE=$($MYSQL "SHOW STATUS LIKE 'wsrep_local_state_comment'" | awk '{print $2}')
FLOW_CONTROL=$($MYSQL "SHOW STATUS LIKE 'wsrep_flow_control_paused'" | awk '{print $2}')

if [ "$CLUSTER_SIZE" -lt 3 ] || [ "$CLUSTER_STATUS" != "Primary" ] || [ "$NODE_STATE" != "Synced" ]; then
    echo "CRITICAL: Galera cluster degradat"
    echo "  Size: $CLUSTER_SIZE | Status: $CLUSTER_STATUS | State: $NODE_STATE | FC: $FLOW_CONTROL"
    exit 2
fi

echo "OK: Galera cluster sănătos (3 noduri, Primary, Synced)"
exit 0
```

---

## Problema split-brain: de ce trei noduri și nu două

Când am prezentat soluția clientului, prima întrebare a fost: "Chiar avem nevoie de trei servere? Nu sunt suficiente două?"

Nu. Și nu e o chestiune de cost — e o chestiune de matematică.

Galera folosește un algoritm de consens bazat pe **quorum**. Cu trei noduri, quorum-ul este 2: dacă un nod cade, celelalte două recunosc că sunt majoritatea și continuă să funcționeze. Cu două noduri, quorum-ul este 2: dacă un nod cade, cel rămas **nu are quorum** și se blochează pentru a preveni split-brain-ul.

Parametrul `pc.ignore_quorum` există pentru a forța un nod să funcționeze fără quorum, dar e ca și cum ai dezactiva alarma de incendiu pentru că sună prea des.

**Trei noduri este minimul pentru un cluster Galera în producție.** Al treilea nod nu este un lux — este ceea ce permite clusterului să continue să funcționeze când lucrurile merg prost.

---

## Când un nod cade și revine

Unul dintre primele lucruri pe care le-am făcut după punerea în producție a fost să simulez o defecțiune — cu clientul urmărind.

Am oprit Nodul 3. Aplicația a continuat să funcționeze fără întrerupere pe nodurile 1 și 2. Fără erori, fără timeout-uri. Două sute de utilizatori care nu au observat nimic.

Apoi am repornit Nodul 3. Ce s-a întâmplat:

1. Nodul a pornit și a contactat celelalte prin `wsrep_cluster_address`
2. Gap-ul de tranzacții era mic, deci a primit un **IST** (Incremental State Transfer) — doar tranzacțiile lipsă
3. În mai puțin de un minut era din nou în starea `Synced`

Dacă nodul ar fi rămas oprit mai mult timp și gcache-ul ar fi fost depășit, ar fi primit un **SST** complet — întregul set de date. De aceea parametrul `gcache.size` este important:

```ini
wsrep_provider_options="gcache.size=512M"
```

Un gcache mai mare înseamnă că clusterul poate tolera perioade mai lungi de inactivitate ale unui nod fără a necesita un SST complet. În cazul clientului, cu aproximativ 80-100 MB de tranzacții pe zi, un gcache de 512 MB acoperea aproape o săptămână de absență.

Clientul a privit Nodul 3 revenind în sync și a spus: "Deci data viitoare când trebuie să facem mentenanță pe un server, nu mai trebuie să oprim totul?" Exact. Acesta era scopul.

---

## Checklist de punere în producție

Înainte de a declara clusterul gata pentru client, am verificat fiecare punct:

- [ ] Aceeași versiune MariaDB pe toate nodurile
- [ ] `wsrep_cluster_size` = 3
- [ ] `wsrep_cluster_status` = Primary pe toate nodurile
- [ ] `wsrep_local_state_comment` = Synced pe toate nodurile
- [ ] Test de scriere pe Nodul 1, verificare de citire pe Nodul 2 și 3
- [ ] Test de oprire a unui nod: clusterul continuă să funcționeze
- [ ] Test de rejoin: nodul revine la Synced fără SST complet
- [ ] Utilizator SST configurat și funcțional
- [ ] Firewall verificat pe toate porturile (3306, 4567, 4568, 4444)
- [ ] Monitorizare activă pe `wsrep_cluster_status` și `wsrep_flow_control_paused`
- [ ] Backup configurat (pe UN singur nod, nu pe toate trei)
- [ ] Aplicația reconfigurată să indice către load balancer sau VIP

---

## Șase luni mai târziu

Am aflat de la client la șase luni după punerea în producție. Între timp avuseseră două restartări programate pentru actualizări de sistem și o defecțiune neprevăzută a unui disc pe unul dintre noduri. În toate cele trei cazuri, aplicația nu a încetat niciodată să funcționeze. Zero downtime neplanificat.

Ce m-a impresionat cel mai mult a fost fraza lui: "Înainte trăiam cu anxietatea că baza de date va cădea. Acum nu ne mai gândim la asta."

Aceasta este adevărata valoare a unui cluster Galera bine configurat. Nu este tehnologia în sine — este liniștea pe care o aduce. Certitudinea că o singură defecțiune nu mai oprește compania.

Partea tehnică este cea mai simplă. Ceea ce face diferența este înțelegerea **de ce** fiecare parametru este setat într-un anumit fel, ce se întâmplă când lucrurile merg prost și cum să diagnostichezi problemele înainte ca ele să devină urgențe. Un cluster care funcționează în demo și unul care rezistă în producție: distanța dintre cele două stă toată în detaliile pe care le-am povestit aici.
