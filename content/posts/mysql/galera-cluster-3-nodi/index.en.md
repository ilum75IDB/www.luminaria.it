---
title: "Galera Cluster with 3 nodes: how I solved a MySQL availability problem"
description: "A client with a standalone MySQL that crashed every month, taking down the entire application. My solution: a 3-node Galera Cluster with synchronous replication. From diagnosis to production, with all configuration files and critical parameters."
date: "2026-02-17T10:00:00+01:00"
draft: false
translationKey: "galera_cluster_3_nodi"
tags: ["mariadb", "galera", "cluster", "high-availability", "replication", "wsrep"]
categories: ["mysql"]
image: "galera-cluster-3-nodi.cover.jpg"
---

The ticket was laconic, as it often is when the problem is serious: "The database went down again. The application is stopped. Third time in two months."

The client had a MariaDB on a single Linux server — a business management application used by about two hundred internal users, with load spikes during end-of-month accounting closures. Every time the server had a problem — a disk slowing down, a system update requiring a reboot, a process consuming all the RAM — the database crashed and with it the entire business operations.

The question wasn't "how do we fix the server". The question was: **how do we make sure that the next time a server has a problem, the application keeps running?**

The answer, after twenty years of experience with this type of scenario, was one: **Galera Cluster**.

---

## The diagnosis: a classic single point of failure

The first thing I did was analyse the infrastructure. The picture was familiar:

- A single MariaDB server, no replica
- Nightly backup to external disk (at least that)
- No failover mechanism
- The application pointed directly to the database server's IP

Every downtime, even ten minutes, meant two hundred people idle. During accounting closures, it meant delays cascading across business processes.

I proposed a solution based on Galera Cluster: three MariaDB nodes with synchronous multi-master replication. Any node accepts reads and writes, data is consistent across all three, and if one node goes down the other two continue serving the application without interruption.

The client already had three Linux VMs available — the infrastructure team had provisioned them for another project that was later postponed. Perfect: no need to even order hardware.

---

## The plan: three nodes, zero single point of failure

The available machines:

| Node | Hostname | IP |
|------|----------|-----|
| Node 1 | `db-node1` | `10.0.1.11` |
| Node 2 | `db-node2` | `10.0.1.12` |
| Node 3 | `db-node3` | `10.0.1.13` |

An important premise: **Galera is not a native option in MySQL Community**. You either use MariaDB (which integrates it natively) or Percona XtraDB Cluster (based on MySQL). The client was already using MariaDB, so the choice was natural and required no engine migration.

The goal was clear: migrate from a single-node architecture to a three-node cluster, without changing the application beyond the connection address.

---

## Installation: same version on all nodes

First non-negotiable principle: **all nodes must have the exact same version of MariaDB**. I've seen clusters unstable for months because someone had updated one node and not the others.

On all three servers:

```bash
# Add the official MariaDB repository (example for 11.4 LTS)
curl -LsS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | \
    sudo bash -s -- --mariadb-server-version="mariadb-11.4"

# Install MariaDB Server and the Galera plugin
sudo dnf install MariaDB-server MariaDB-client galera-4 -y

# Enable but DO NOT start the service yet
sudo systemctl enable mariadb
```

Don't start the service. Configure first. Always.

---

## The heart of configuration: `/etc/my.cnf.d/galera.cnf`

This file defines the cluster's behaviour. It must be created on every node, with the appropriate differences for IP address and node name.

Here's the complete configuration for **Node 1**:

```ini
[mysqld]
# === Engine and charset ===
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
innodb_flush_log_at_trx_commit=2
innodb_buffer_pool_size=1G

# === {{< glossary term="wsrep" >}}WSREP{{< /glossary >}} (Galera) configuration ===
wsrep_on=ON
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so

# List of ALL cluster nodes
wsrep_cluster_address="gcomm://10.0.1.11,10.0.1.12,10.0.1.13"

# Cluster name (must be identical on all nodes)
wsrep_cluster_name="galera_production"

# THIS node's identity (changes on each server)
wsrep_node_address="10.0.1.11"
wsrep_node_name="db-node1"

# SST method (State Snapshot Transfer)
wsrep_sst_method=mariabackup
wsrep_sst_auth="sst_user:secure_password_here"

# === Network ===
bind-address=0.0.0.0
```

For **Node 2** and **Node 3**, the only things that change are:

```ini
# Node 2
wsrep_node_address="10.0.1.12"
wsrep_node_name="db-node2"

# Node 3
wsrep_node_address="10.0.1.13"
wsrep_node_name="db-node3"
```

Everything else is identical. **Identical**. Don't give in to the temptation to "customise" buffer pool or other parameters per node: in a Galera cluster, symmetry is a virtue.

---

## Why every parameter matters

Let's go through the parameters one by one, because each has a precise reason.

### `binlog_format=ROW`

Galera requires ROW format for the binary log. Not STATEMENT, not MIXED. **ROW** only. With other formats the cluster won't even start — and rightly so, because synchronous replication based on certification depends on row-level precision.

### `innodb_autoinc_lock_mode=2`

This sets the lock mode for auto-increment to "interleaved". In a multi-master cluster, two nodes can generate INSERTs simultaneously on the same table. With lock mode 1 (the default) this would create deadlocks. With value 2, InnoDB generates auto-increments without a global lock, allowing concurrent inserts from different nodes.

The consequence: auto-increment IDs **won't be sequential** across nodes. If your application depends on sequential IDs, you have an architectural problem to solve upstream.

### `innodb_flush_log_at_trx_commit=2`

Here we make a conscious trade-off. Value 1 (default) guarantees total durability — every commit is written and fsynced to disk. But in a Galera cluster, durability is already guaranteed by synchronous replication across three nodes. Value 2 writes to the OS buffer on each commit and fsyncs only every second, improving write performance by 30-40% in our tests.

If you lose one node, the data is on the other two. If you lose the entire datacenter... well, that's another conversation.

### `wsrep_sst_method=mariabackup`

{{< glossary term="sst" >}}SST{{< /glossary >}} is the mechanism by which a node joining the cluster receives a complete copy of the data. The options are:

| Method | Pro | Con |
|--------|-----|-----|
| `rsync` | Fast | Donor node blocks on reads |
| `mariabackup` | Doesn't block the donor | Requires separate installation |
| `mysqldump` | Simple | Very slow, blocks the donor |

**Always mariabackup**. In production, blocking a donor node during an SST means degrading the cluster at the moment you need it most.

```bash
# Install mariabackup on all nodes
sudo dnf install MariaDB-backup -y
```

---

## Firewall: the ports Galera needs open

This is where I see 50% of first installations fail. Galera doesn't just use the MySQL port.

```bash
# On all three nodes
sudo firewall-cmd --permanent --add-port=3306/tcp   # Standard MySQL
sudo firewall-cmd --permanent --add-port=4567/tcp   # Galera cluster communication
sudo firewall-cmd --permanent --add-port=4567/udp   # Multicast replication (optional)
sudo firewall-cmd --permanent --add-port=4568/tcp   # IST (Incremental State Transfer)
sudo firewall-cmd --permanent --add-port=4444/tcp   # SST (State Snapshot Transfer)
sudo firewall-cmd --reload
```

If SELinux is active (and it should be), you also need the policies:

```bash
sudo setsebool -P mysql_connect_any 1
```

Four ports. Four. Not one more, not one less. If you forget one, the cluster forms but doesn't synchronise — and debugging becomes an exercise in frustration.

---

## Data migration and the bootstrap

Before starting the cluster, I migrated the data from the standalone server to Node 1 with a full dump:

```bash
# On the old standalone server
mysqldump --all-databases --single-transaction --routines --triggers \
    --events > /tmp/full_dump.sql

# Transfer to Node 1
scp /tmp/full_dump.sql db-node1:/tmp/
```

Then the bootstrap — the moment of truth. The first node doesn't start with `systemctl start mariadb`. It starts with the bootstrap command.

**Only on Node 1:**

```bash
sudo galera_new_cluster
```

This command starts MariaDB with `wsrep_cluster_address=gcomm://` (empty), which means: "I am the founder, I'm not looking for other nodes."

Data import and SST user creation:

```sql
-- Import the dump from the old server
SOURCE /tmp/full_dump.sql;

-- Create the user for data transfer between nodes
CREATE USER 'sst_user'@'localhost' IDENTIFIED BY 'secure_password_here';
GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'sst_user'@'localhost';
FLUSH PRIVILEGES;
```

Immediate verification:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | wsrep_cluster_size | 1     |
-- +--------------------+-------+
```

If the value is 1, the bootstrap worked. Now, **on the other two nodes:**

```bash
sudo systemctl start mariadb
```

These nodes read `wsrep_cluster_address`, find Node 1, receive a full SST with all the data and join the cluster.

After starting all three:

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';
-- +--------------------+-------+
-- | Variable_name      | Value |
-- +--------------------+-------+
-- | wsrep_cluster_size | 3     |
-- +--------------------+-------+
```

Three. That's the magic number. The moment the client stopped having a single point of failure.

---

## Checking cluster health

This is the part that truly matters for whoever manages the cluster day to day. It's not enough to know that `wsrep_cluster_size` is 3. You need to read the full status.

### The diagnostic query I always use

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

### How to interpret the results

| Variable | Healthy value | Meaning |
|----------|--------------|---------|
| `wsrep_cluster_size` | `3` | All nodes are in the cluster |
| `wsrep_cluster_status` | `Primary` | The cluster is operational and has quorum |
| `wsrep_connected` | `ON` | This node is connected to the cluster |
| `wsrep_ready` | `ON` | This node accepts queries |
| `wsrep_local_state_comment` | `Synced` | This node is synchronised |
| `wsrep_flow_control_paused` | `0.0` | No flow control pauses |
| `wsrep_local_recv_queue_avg` | `< 0.5` | Receive queue is under control |
| `wsrep_local_send_queue_avg` | `< 0.5` | Send queue is under control |

### Warning signs

**`wsrep_cluster_status = Non-Primary`**: the node has lost quorum. It's isolated. It won't accept writes (and shouldn't). This happens when a node loses connection with the majority of the cluster.

**`wsrep_flow_control_paused > 0.0`**: flow control activated. It means a node is too slow applying transactions and is asking the others to slow down. A value close to 1.0 means the cluster is essentially stalled, waiting for the slowest node.

**`wsrep_local_recv_queue_avg > 1.0`**: incoming transactions are piling up. Could be a disk I/O problem, CPU, or an undersized node.

### Monitoring script

I also delivered a script for their monitoring system (Zabbix, in their case):

```bash
#!/bin/bash
# galera_health_check.sh — run on every node

MYSQL="mysql -u monitor -p'monitor_pwd' -Bse"

CLUSTER_SIZE=$($MYSQL "SHOW STATUS LIKE 'wsrep_cluster_size'" | awk '{print $2}')
CLUSTER_STATUS=$($MYSQL "SHOW STATUS LIKE 'wsrep_cluster_status'" | awk '{print $2}')
NODE_STATE=$($MYSQL "SHOW STATUS LIKE 'wsrep_local_state_comment'" | awk '{print $2}')
FLOW_CONTROL=$($MYSQL "SHOW STATUS LIKE 'wsrep_flow_control_paused'" | awk '{print $2}')

if [ "$CLUSTER_SIZE" -lt 3 ] || [ "$CLUSTER_STATUS" != "Primary" ] || [ "$NODE_STATE" != "Synced" ]; then
    echo "CRITICAL: Galera cluster degraded"
    echo "  Size: $CLUSTER_SIZE | Status: $CLUSTER_STATUS | State: $NODE_STATE | FC: $FLOW_CONTROL"
    exit 2
fi

echo "OK: Galera cluster healthy (3 nodes, Primary, Synced)"
exit 0
```

---

## The {{< glossary term="split-brain" >}}split-brain{{< /glossary >}} problem: why three nodes and not two

When I presented the solution to the client, the first question was: "Do we really need three servers? Wouldn't two be enough?"

No. And it's not a cost issue — it's a matter of mathematics.

Galera uses a consensus algorithm based on {{< glossary term="quorum" >}}**quorum**{{< /glossary >}}. With three nodes, the quorum is 2: if one node fails, the other two recognise they are the majority and continue operating. With two nodes, the quorum is 2: if one node fails, the remaining one **doesn't have quorum** and blocks to prevent split-brain.

The parameter `pc.ignore_quorum` exists to force a node to operate without quorum, but that's like disabling the fire alarm because it rings too often.

**Three nodes is the minimum for a production Galera cluster.** The third node isn't a luxury — it's what allows the cluster to keep running when things go wrong.

---

## When a node goes down and comes back

One of the first things I did after going to production was simulate a failure — with the client watching.

I shut down Node 3. The application kept running without interruption on nodes 1 and 2. No errors, no timeouts. Two hundred users who noticed nothing.

Then I restarted Node 3. What happened:

1. The node started and contacted the others via `wsrep_cluster_address`
2. The transaction gap was small, so it received an {{< glossary term="ist" >}}**IST** (Incremental State Transfer){{< /glossary >}} — only the missing transactions
3. In less than a minute it was `Synced` again

If the node had stayed down longer and the gcache had been exceeded, it would have received a full **SST** — the entire dataset. That's why the `gcache.size` parameter matters:

```ini
wsrep_provider_options="gcache.size=512M"
```

A larger gcache means the cluster can tolerate longer node downtime without requiring a full SST. In the client's case, with about 80-100 MB of transactions per day, a 512 MB gcache covered nearly a week of absence.

The client watched Node 3 come back in sync and said: "So the next time we need to do maintenance on a server, we don't have to stop everything?" Exactly. That was the point.

---

## Production readiness checklist

Before declaring the cluster ready to the client, I verified every point:

- [ ] Same MariaDB version on all nodes
- [ ] `wsrep_cluster_size` = 3
- [ ] `wsrep_cluster_status` = Primary on all nodes
- [ ] `wsrep_local_state_comment` = Synced on all nodes
- [ ] Write test on Node 1, read verification on Node 2 and 3
- [ ] Shutdown test on one node: the cluster keeps running
- [ ] Rejoin test: the node returns to Synced without a full SST
- [ ] SST user configured and working
- [ ] Firewall verified on all ports (3306, 4567, 4568, 4444)
- [ ] Monitoring active on `wsrep_cluster_status` and `wsrep_flow_control_paused`
- [ ] Backup configured (on ONE node, not all three)
- [ ] Application reconfigured to point to the load balancer or VIP

---

## Six months later

I heard from the client six months after going to production. In the meantime they had two scheduled reboots for system updates and one unexpected disk failure on one of the nodes. In all three cases, the application never stopped working. Zero unplanned downtime.

What struck me most was his comment: "We used to live with the anxiety of the database going down. Now we don't think about it anymore."

That's the real value of a well-configured Galera cluster. It's not the technology itself — it's the peace of mind it brings. The certainty that a single failure no longer stops the business.

The technical part is the easiest. What makes the difference is understanding **why** each parameter is set a certain way, what happens when things go wrong, and how to diagnose problems before they become emergencies. A cluster that works in a demo and one that holds in production: the distance between the two is all in the details I've described here.

------------------------------------------------------------------------

## Glossary

**[Quorum](/en/glossary/quorum/)** — Majority-based consensus mechanism. With 3 nodes the quorum is 2: if one fails, the other two continue operating. This is what prevents split-brain.

**[SST](/en/glossary/sst/)** — State Snapshot Transfer: mechanism by which a node joining the cluster receives a complete copy of the entire dataset from a donor node. The recommended method is mariabackup.

**[IST](/en/glossary/ist/)** — Incremental State Transfer: transfer of only the missing transactions to a node rejoining the cluster after a brief absence. Much faster than a full SST.

**[WSREP](/en/glossary/wsrep/)** — Write Set Replication: synchronous replication API and protocol used by Galera Cluster. Each transaction is replicated to all nodes before commit through a certification process.

**[Split-brain](/en/glossary/split-brain/)** — Critical condition where two parts of the cluster operate independently accepting divergent writes. Quorum prevents it: only the partition with the majority of nodes can continue operating.
