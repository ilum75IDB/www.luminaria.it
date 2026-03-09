---
title: "Galera Cluster a 3 nodi: come ho risolto un problema di disponibilità su MySQL"
description: "Un cliente con un MySQL standalone che cadeva ogni mese portando giù l'intera applicazione. La mia soluzione: un Galera Cluster a 3 nodi con replica sincrona. Dalla diagnosi alla messa in produzione, con tutti i file di configurazione e i parametri critici."
date: "2026-02-24T10:00:00+01:00"
draft: false
translationKey: "galera_cluster_3_nodi"
tags: ["mysql", "mariadb", "galera", "cluster", "alta disponibilità", "replica", "wsrep", "linux"]
categories: ["mysql"]
image: "galera-cluster-3-nodi.cover.jpg"
---

Il ticket era laconico, come spesso accade quando il problema è grave: "Il database è andato giù di nuovo. L'applicazione è ferma. Terza volta in due mesi."

Il cliente aveva un MariaDB su un singolo server Linux — un'applicazione gestionale usata da circa duecento utenti interni, con picchi di carico durante le chiusure contabili di fine mese. Ogni volta che il server aveva un problema — un disco che rallentava, un aggiornamento di sistema che richiedeva un riavvio, un processo che consumava tutta la RAM — il database cadeva e con lui l'intera operatività aziendale.

La domanda non era "come ripariamo il server". La domanda era: **come facciamo in modo che la prossima volta che un server ha un problema, l'applicazione continui a funzionare?**

La risposta, dopo vent'anni di esperienza con questo tipo di scenari, era una: **Galera Cluster**.

---

## La diagnosi: un single point of failure classico

La prima cosa che ho fatto è stata analizzare l'infrastruttura. Il quadro era familiare:

- Un singolo server MariaDB, nessuna replica
- Backup notturno su disco esterno (almeno quello)
- Nessun meccanismo di failover
- L'applicazione puntava direttamente all'IP del server database

Ogni downtime, anche di dieci minuti, significava duecento persone ferme. Durante le chiusure contabili, significava ritardi che si propagavano a cascata sui processi aziendali.

Ho proposto al cliente una soluzione basata su Galera Cluster: tre nodi MariaDB con replica sincrona multi-master. Qualsiasi nodo accetta letture e scritture, i dati sono coerenti su tutti e tre, e se un nodo cade gli altri due continuano a servire l'applicazione senza interruzione.

Il cliente aveva già a disposizione tre VM Linux — il team infrastruttura le aveva provisioned per un altro progetto poi rimandato. Perfetto: non serviva nemmeno ordinare hardware.

---

## Il piano: tre nodi, zero single point of failure

Le macchine a disposizione:

| Nodo | Hostname | IP |
|------|----------|-----|
| Nodo 1 | `db-node1` | `10.0.1.11` |
| Nodo 2 | `db-node2` | `10.0.1.12` |
| Nodo 3 | `db-node3` | `10.0.1.13` |

Una premessa importante: **Galera non è un'opzione nativa di MySQL Community**. Si usa MariaDB (che lo integra nativamente) oppure Percona XtraDB Cluster (basato su MySQL). Il cliente usava già MariaDB, quindi la scelta era naturale e non richiedeva migrazione di engine.

L'obiettivo era chiaro: migrare da un'architettura a singolo nodo a un cluster a tre nodi, senza cambiare l'applicazione se non per l'indirizzo di connessione.

---

## Installazione: stessa versione su tutti i nodi

Primo principio non negoziabile: **tutti i nodi devono avere la stessa identica versione di MariaDB**. Ho visto cluster instabili per mesi perché qualcuno aveva aggiornato un nodo e non gli altri.

Su tutti e tre i server:

```bash
# Aggiungere il repository ufficiale MariaDB (esempio per 11.4 LTS)
curl -LsS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | \
    sudo bash -s -- --mariadb-server-version="mariadb-11.4"

# Installare MariaDB Server e il plugin Galera
sudo dnf install MariaDB-server MariaDB-client galera-4 -y

# Abilitare ma NON avviare ancora il servizio
sudo systemctl enable mariadb
```

Non avviare il servizio. Prima si configura. Sempre.

---

## Il cuore della configurazione: `/etc/my.cnf.d/galera.cnf`

Questo è il file che definisce il comportamento del cluster. Va creato su ogni nodo, con le dovute differenze per indirizzo IP e nome nodo.

Ecco la configurazione completa per il **Nodo 1**:

```ini
[mysqld]
# === Motore e charset ===
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
innodb_flush_log_at_trx_commit=2
innodb_buffer_pool_size=1G

# === Configurazione WSREP (Galera) ===
wsrep_on=ON
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so

# Lista di TUTTI i nodi del cluster
wsrep_cluster_address="gcomm://10.0.1.11,10.0.1.12,10.0.1.13"

# Nome del cluster (deve essere identico su tutti i nodi)
wsrep_cluster_name="galera_produzione"

# Identità di QUESTO nodo (cambia su ogni server)
wsrep_node_address="10.0.1.11"
wsrep_node_name="db-node1"

# Metodo SST (State Snapshot Transfer)
wsrep_sst_method=mariabackup
wsrep_sst_auth="sst_user:password_sicura_qui"

# === Rete ===
bind-address=0.0.0.0
```

Per il **Nodo 2** e il **Nodo 3**, l'unica cosa che cambia è:

```ini
# Nodo 2
wsrep_node_address="10.0.1.12"
wsrep_node_name="db-node2"

# Nodo 3
wsrep_node_address="10.0.1.13"
wsrep_node_name="db-node3"
```

Tutto il resto è identico. **Identico**. Non cedere alla tentazione di "personalizzare" buffer pool o altri parametri per nodo: in un cluster Galera, la simmetria è una virtù.

---

## Perché ogni parametro conta

Vediamo i parametri uno per uno, perché ognuno ha una ragione precisa.

### `binlog_format=ROW`

Galera richiede il formato ROW per il binary log. Non STATEMENT, non MIXED. **ROW** e basta. Con gli altri formati il cluster non si avvia nemmeno — e giustamente, perché la replica sincrona basata su certification dipende dalla precisione a livello di riga.

### `innodb_autoinc_lock_mode=2`

Questo parametro imposta il lock mode per gli auto-increment a "interleaved". In un cluster multi-master, due nodi possono generare INSERT contemporaneamente sulla stessa tabella. Con il lock mode 1 (il default) si creerebbero deadlock. Con il valore 2, InnoDB genera gli auto-increment senza lock globale, permettendo inserimenti concorrenti da nodi diversi.

La conseguenza: gli ID auto-increment **non saranno sequenziali** tra i nodi. Se la tua applicazione dipende dalla sequenzialità degli ID, hai un problema architetturale da risolvere a monte.

### `innodb_flush_log_at_trx_commit=2`

Qui si fa un compromesso consapevole. Il valore 1 (default) garantisce durabilità totale — ogni commit viene scritto e fsynced sul disco. Ma in un cluster Galera, la durabilità è già garantita dalla replica sincrona su tre nodi. Il valore 2 scrive nel buffer del sistema operativo ad ogni commit e fa fsync solo ogni secondo, migliorando le performance di scrittura del 30-40% nei nostri test.

Se perdi un nodo, i dati sono sugli altri due. Se perdi il datacenter intero... beh, quello è un altro discorso.

### `wsrep_sst_method=mariabackup`

SST è il meccanismo con cui un nodo che si unisce al cluster riceve una copia completa dei dati. Le opzioni sono:

| Metodo | Pro | Contro |
|--------|-----|--------|
| `rsync` | Veloce | Il nodo donatore si blocca in lettura |
| `mariabackup` | Non blocca il donatore | Richiede installazione separata |
| `mysqldump` | Semplice | Lentissimo, blocca il donatore |

**Sempre mariabackup**. In produzione, bloccare un nodo donatore durante un SST significa degradare il cluster nel momento in cui ne hai più bisogno.

```bash
# Installare mariabackup su tutti i nodi
sudo dnf install MariaDB-backup -y
```

---

## Firewall: le porte che Galera vuole aperte

Questo è il punto dove vedo fallire il 50% delle prime installazioni. Galera non usa solo la porta MySQL.

```bash
# Su tutti e tre i nodi
sudo firewall-cmd --permanent --add-port=3306/tcp   # MySQL standard
sudo firewall-cmd --permanent --add-port=4567/tcp   # Comunicazione cluster Galera
sudo firewall-cmd --permanent --add-port=4567/udp   # Multicast replication (opzionale)
sudo firewall-cmd --permanent --add-port=4568/tcp   # IST (Incremental State Transfer)
sudo firewall-cmd --permanent --add-port=4444/tcp   # SST (State Snapshot Transfer)
sudo firewall-cmd --reload
```

Se SELinux è attivo (e dovrebbe esserlo), servono anche le policy:

```bash
sudo setsebool -P mysql_connect_any 1
```

Quattro porte. Quattro. Non una di più, non una di meno. Se ne dimentichi una, il cluster si forma ma non sincronizza — e il debug diventa un esercizio di frustrazione.

---

## La migrazione dei dati e il bootstrap

Prima di avviare il cluster, ho migrato i dati dal server standalone al Nodo 1 con un dump completo:

```bash
# Sul vecchio server standalone
mysqldump --all-databases --single-transaction --routines --triggers \
    --events > /tmp/full_dump.sql

# Trasferimento sul Nodo 1
scp /tmp/full_dump.sql db-node1:/tmp/
```

Poi il bootstrap — il momento della verità. Il primo nodo non si avvia con `systemctl start mariadb`. Si avvia con il comando di bootstrap.

**Solo sul Nodo 1:**

```bash
sudo galera_new_cluster
```

Questo comando avvia MariaDB con `wsrep_cluster_address=gcomm://` (vuoto), che significa: "Io sono il fondatore, non cerco altri nodi."

Importazione dei dati e creazione dell'utente SST:

```sql
-- Importare il dump dal vecchio server
SOURCE /tmp/full_dump.sql;

-- Creare l'utente per il trasferimento dati tra nodi
CREATE USER 'sst_user'@'localhost' IDENTIFIED BY 'password_sicura_qui';
GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sst_user'@'localhost';
FLUSH PRIVILEGES;
```

Verifica immediata:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | wsrep_cluster_size | 1     |
-- +--------------------+-------+
```

Se il valore è 1, il bootstrap ha funzionato. Ora, **sugli altri due nodi:**

```bash
sudo systemctl start mariadb
```

Questi nodi leggono `wsrep_cluster_address`, trovano il Nodo 1, ricevono un SST completo con tutti i dati e si uniscono al cluster.

Dopo aver avviato tutti e tre:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | wsrep_cluster_size | 3     |
-- +--------------------+-------+
```

Tre. Questo è il numero magico. Il momento in cui il cliente ha smesso di avere un single point of failure.

---

## Verificare lo stato di salute del cluster

Questa è la parte che interessa davvero a chi gestisce il cluster giorno per giorno. Non basta sapere che `wsrep_cluster_size` è 3. Bisogna sapere leggere lo stato completo.

### La query diagnostica che uso sempre

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

### Come interpretare i risultati

| Variabile | Valore sano | Cosa significa |
|-----------|-------------|----------------|
| `wsrep_cluster_size` | `3` | Tutti i nodi sono nel cluster |
| `wsrep_cluster_status` | `Primary` | Il cluster è operativo e ha quorum |
| `wsrep_connected` | `ON` | Questo nodo è connesso al cluster |
| `wsrep_ready` | `ON` | Questo nodo accetta query |
| `wsrep_local_state_comment` | `Synced` | Questo nodo è sincronizzato |
| `wsrep_flow_control_paused` | `0.0` | Nessuna pausa per flow control |
| `wsrep_local_recv_queue_avg` | `< 0.5` | La coda di ricezione è sotto controllo |
| `wsrep_local_send_queue_avg` | `< 0.5` | La coda di invio è sotto controllo |

### I segnali di pericolo

**`wsrep_cluster_status = Non-Primary`**: il nodo ha perso il quorum. È isolato. Non accetterà scritture (e non deve). Questo succede quando un nodo perde la connessione con la maggioranza del cluster.

**`wsrep_flow_control_paused > 0.0`**: flow control attivato. Significa che un nodo è troppo lento nell'applicare le transazioni e sta chiedendo agli altri di rallentare. Un valore vicino a 1.0 significa che il cluster è sostanzialmente fermo, in attesa del nodo più lento.

**`wsrep_local_recv_queue_avg > 1.0`**: le transazioni in arrivo si accumulano. Potrebbe essere un problema di I/O disco, CPU, o un nodo sottodimensionato.

### Script di monitoraggio

Ho consegnato al cliente anche uno script per il loro sistema di monitoring (Zabbix, nel loro caso):

```bash
#!/bin/bash
# galera_health_check.sh — da eseguire su ogni nodo

MYSQL="mysql -u monitor -p'monitor_pwd' -Bse"

CLUSTER_SIZE=$($MYSQL "SHOW STATUS LIKE 'wsrep_cluster_size'" | awk '{print $2}')
CLUSTER_STATUS=$($MYSQL "SHOW STATUS LIKE 'wsrep_cluster_status'" | awk '{print $2}')
NODE_STATE=$($MYSQL "SHOW STATUS LIKE 'wsrep_local_state_comment'" | awk '{print $2}')
FLOW_CONTROL=$($MYSQL "SHOW STATUS LIKE 'wsrep_flow_control_paused'" | awk '{print $2}')

if [ "$CLUSTER_SIZE" -lt 3 ] || [ "$CLUSTER_STATUS" != "Primary" ] || [ "$NODE_STATE" != "Synced" ]; then
    echo "CRITICAL: Galera cluster degradato"
    echo "  Size: $CLUSTER_SIZE | Status: $CLUSTER_STATUS | State: $NODE_STATE | FC: $FLOW_CONTROL"
    exit 2
fi

echo "OK: Galera cluster sano (3 nodi, Primary, Synced)"
exit 0
```

---

## Il problema dello split-brain: perché tre nodi e non due

Quando ho presentato la soluzione al cliente, la prima domanda è stata: "Servono davvero tre server? Non bastano due?"

No. E non è una questione di costo — è una questione di matematica.

Galera usa un algoritmo di consenso basato su **quorum**. Con tre nodi, il quorum è 2: se un nodo cade, gli altri due riconoscono di essere la maggioranza e continuano a operare. Con due nodi, il quorum è 2: se un nodo cade, quello rimasto **non ha quorum** e si blocca per evitare lo split-brain.

Esiste il parametro `pc.ignore_quorum` per forzare un nodo a operare senza quorum, ma è come disattivare l'allarme antincendio perché suona troppo spesso.

**Tre nodi è il minimo per un cluster Galera in produzione.** Il terzo nodo non è un lusso — è quello che permette al cluster di continuare a funzionare quando le cose vanno male.

---

## Quando un nodo va giù e torna su

Una delle prime cose che ho fatto dopo la messa in produzione è stata simulare un guasto — con il cliente che guardava.

Ho spento il Nodo 3. L'applicazione ha continuato a funzionare senza interruzione sui nodi 1 e 2. Nessun errore, nessun timeout. Duecento utenti che non si sono accorti di nulla.

Poi ho riavviato il Nodo 3. Cosa è successo:

1. Il nodo si è avviato e ha contattato gli altri tramite `wsrep_cluster_address`
2. Il gap di transazioni era piccolo, quindi ha ricevuto un **IST** (Incremental State Transfer) — solo le transazioni mancanti
3. In meno di un minuto era di nuovo `Synced`

Se il nodo fosse rimasto giù più a lungo e il gcache fosse stato superato, avrebbe ricevuto un **SST** completo — l'intero dataset. Per questo il parametro `gcache.size` è importante:

```ini
wsrep_provider_options="gcache.size=512M"
```

Un gcache più grande significa che il cluster può tollerare downtime più lunghi di un nodo senza dover fare un SST completo. Nel caso del cliente, con circa 80-100 MB di transazioni al giorno, un gcache di 512 MB copriva quasi una settimana di assenza.

Il cliente ha guardato il Nodo 3 tornare in sync e ha detto: "Quindi la prossima volta che dobbiamo fare manutenzione su un server, non dobbiamo più fermare tutto?" Esatto. Quello era il punto.

---

## Checklist di messa in produzione

Prima di dichiarare il cluster pronto al cliente, ho verificato ogni punto:

- [ ] Stessa versione MariaDB su tutti i nodi
- [ ] `wsrep_cluster_size` = 3
- [ ] `wsrep_cluster_status` = Primary su tutti i nodi
- [ ] `wsrep_local_state_comment` = Synced su tutti i nodi
- [ ] Test di scrittura su Nodo 1, verifica lettura su Nodo 2 e 3
- [ ] Test di shutdown di un nodo: il cluster continua a funzionare
- [ ] Test di rejoin: il nodo torna in Synced senza SST completo
- [ ] Utente SST configurato e funzionante
- [ ] Firewall verificato su tutte le porte (3306, 4567, 4568, 4444)
- [ ] Monitoring attivo su `wsrep_cluster_status` e `wsrep_flow_control_paused`
- [ ] Backup configurato (su UN nodo, non su tutti e tre)
- [ ] Applicazione riconfigurata per puntare al load balancer o al VIP

---

## Sei mesi dopo

Ho risentito il cliente sei mesi dopo la messa in produzione. Nel frattempo avevano avuto due riavvii programmati per aggiornamenti di sistema e un guasto imprevisto a un disco su uno dei nodi. In tutti e tre i casi, l'applicazione non ha mai smesso di funzionare. Zero downtime non pianificato.

La cosa che mi ha colpito di più è stata la sua frase: "Prima vivevamo con l'ansia del database che cadeva. Adesso non ci pensiamo più."

Questo è il vero valore di un cluster Galera ben configurato. Non è la tecnologia in sé — è la tranquillità che porta. La certezza che un singolo guasto non ferma più l'azienda.

La parte tecnica è la più semplice. Quello che fa la differenza è capire **perché** ogni parametro è impostato in un certo modo, cosa succede quando le cose vanno male, e come diagnosticare i problemi prima che diventino emergenze. Un cluster che funziona in demo e uno che regge in produzione: la distanza tra i due è tutta nei dettagli che ho raccontato qui.
