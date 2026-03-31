---
title: "Full disk on a MySQL cluster: binary logs, Group Replication, and a migration that leaves no room for mistakes"
description: "Filesystem at 92% on a 3-node MySQL Group Replication cluster. The culprit? Binary logs piling up on the main volume. From the initial alert to a dedicated volume migration, one node at a time, without losing quorum."
date: "2025-10-14T08:03:00+01:00"
draft: false
translationKey: "mysql_group_replication_binlog_migration"
tags: ["group-replication", "binary-log", "disk-space", "cluster", "innodb-cluster"]
categories: ["mysql"]
image: "mysql-group-replication-binlog-migration.cover.jpg"
---

The alert came on a Monday morning, wedged between three meetings and a coffee that was still hot. "Filesystem /mysql at 85% on the primary node." On another node it was 66%, on the third 25%. In a cluster, when the numbers don't match across nodes, there's always something going on underneath.

The first question that comes to mind is "how much space do we need?" But that's the wrong question. The right one is: "why is it filling up?"

---

## The cause: binary logs on the wrong volume

Checking was quick:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

Result: `ON`. Binary logs were active — as expected in a cluster. But the path was the issue:

```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```

```
/mysql/bin_log/binlog
```

The binlogs were sitting on the same volume as the data: `/mysql`. A roughly 3 TB volume that on one node was already at 85%.

I also checked retention:

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

```
604800
```

Seven days. Not an unreasonable value, but with three nodes each writing local binlogs on a volume shared with the data, seven days can add up fast — especially under heavy write load.

The picture was clear: binary logs were eating up the main filesystem. Not a bug, not a runaway table. Just an architectural choice made at installation time and never revisited.

---

## What kind of cluster is this, exactly?

Before touching anything on a MySQL server — before even thinking about moving a file — you need to know what you're dealing with. "It's a cluster" isn't enough. MySQL has at least four different ways of doing high availability, and each one has its own rules.

I started with classic replication:

```sql
SHOW SLAVE STATUS\G
```

Empty set on both nodes I checked. No traditional replication running.

Then I tried `SHOW REPLICA STATUS` — but on MySQL 8.0.20 that command doesn't exist yet. It was introduced in 8.0.22. A detail that online documentation often forgets to mention, leaving you chasing a syntax error that isn't one.

Next step — Group Replication:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

And there it was:

| MEMBER_HOST | MEMBER_STATE | MEMBER_ROLE |
|-------------|-------------|-------------|
| dbcluster01 | ONLINE | SECONDARY |
| dbcluster02 | ONLINE | SECONDARY |
| dbcluster03 | ONLINE | PRIMARY |

Three nodes. All ONLINE. One primary, two secondaries. Group Replication in single-primary mode.

Final confirmation from the plugins:

```sql
SHOW PLUGINS;
```

In the list: `group_replication | ACTIVE | GROUP REPLICATION | group_replication.so`. And from the configuration:

```sql
SHOW VARIABLES LIKE 'group_replication_single_primary_mode';
```

```
ON
```

Now I knew exactly what I was dealing with. Not classic replication, not Galera, not NDB Cluster. A MySQL Group Replication single-primary with three nodes, GTID enabled, ROW-based binlog format. The full picture.

The temptation is always to skip this phase. "I know it's a cluster, let's move." But skipping diagnosis on a cluster is like operating without a CT scan: you might get lucky, or you might cause a disaster.

---

## The solution: a dedicated volume for binary logs

The strategy was straightforward: binlogs need their own volume. Not on the same filesystem as the data, not on an improvised symlink, not on a shared directory. A dedicated volume, mounted at the same path on all three nodes.

I asked the sysadmins to provision a new 600 GB volume with mount point `/mysql/binary_logs` on each of the three nodes.

When the volume was ready, I verified on all three:

```bash
df -h /mysql/binary_logs
```

| Node | /mysql | /mysql/binary_logs |
|------|--------|--------------------|
| dbcluster03 (PRIMARY) | 85% | 1% |
| dbcluster02 (SECONDARY) | 66% | 1% |
| dbcluster01 (SECONDARY) | 25% | 1% |

Fresh, dedicated space. Each volume on a local disk belonging to the respective VM — three disks, three volumes, same mountpoint across all three nodes. The sysadmins had done a clean job.

---

## The checks before touching MySQL

Before stopping the first node, I ran three checks that I consider mandatory.

**Directory permissions.** MySQL won't start if it can't write to the binlog directory. Sounds obvious, but it's one of the most common reasons for "why won't it restart after the config change."

```bash
ls -ld /mysql/binary_logs
```

On all three nodes the permissions were 755. It works, but it's not great security-wise — binlogs can contain sensitive data. I changed them to 750:

```bash
chmod 750 /mysql/binary_logs
```

Result: `drwxr-x--- mysql mysql`. Only the mysql user can read and write.

**Real write test.** Before letting MySQL write there, I verified the filesystem was responding:

```bash
touch /mysql/binary_logs/testfile
ls -l /mysql/binary_logs/testfile
rm -f /mysql/binary_logs/testfile
```

If the touch fails, the problem is storage or permissions — and better to find out now than after a MySQL restart.

**Cluster state.** The last check before proceeding:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

Three nodes ONLINE. Quorum intact. Ready to go.

---

## The strategy: one node at a time, primary last

In a three-node Group Replication, quorum is two. If you stop one node, the other two keep the group alive. If you stop two — you've lost the cluster.

The rule is simple: **one node at a time, waiting for the previous one to rejoin the group before touching the next**. And the primary goes last.

Why? Because when you stop the primary, something important happens: the cluster triggers an automatic election and one of the secondaries becomes the new primary. During those seconds — just a few, if everything is healthy — active connections may be dropped, in-flight transactions may fail. It's a brief disruption, but it's a disruption. It needs to be communicated.

The order I followed:

1. **dbcluster01** (SECONDARY)
2. **dbcluster02** (SECONDARY)
3. **dbcluster03** (PRIMARY)

---

## The procedure, node by node

On each node the sequence is identical:

**A. Verify the node's role.** Before stopping it, confirm it is what you think:

```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```

**B. Stop MySQL:**

```bash
systemctl stop mysqld
```

**C. Modify the configuration.** In `my.cnf`, change the `log_bin` parameter:

From:
```ini
log_bin=/mysql/bin_log/binlog
```

To:
```ini
log_bin=/mysql/binary_logs/mysql-bin
```

One line. One single change. Don't touch the Group Replication parameters, don't change the `server_id`, don't reinvent the engine while you're changing a tyre.

**D. Start MySQL:**

```bash
systemctl start mysqld
```

**E. Verify.** Three things to check:

The new path:
```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```
Must return `/mysql/binary_logs/mysql-bin`.

Cluster membership:
```sql
SELECT MEMBER_HOST, MEMBER_STATE, MEMBER_ROLE
FROM performance_schema.replication_group_members;
```
The node must come back ONLINE.

New binlogs on the new path:
```bash
ls -lh /mysql/binary_logs/
```
New `mysql-bin.000001` files should appear.

**Only when the node is ONLINE and the cluster shows three active nodes again do you move to the next one.** Not before.

For the primary — dbcluster03 — the procedure is identical, but before stopping it I verified that both secondaries were ONLINE and already migrated. At the moment of the stop, the cluster triggered the election. One of the secondaries became primary. Brief disruption, as expected.

---

## What not to do

From my experience, these are the most common traps in this kind of intervention:

**Don't copy old binlogs to the new path.** In Group Replication there's no need for binary archaeology. New binlogs will be created in the new directory after the restart. The old ones are only needed if you require point-in-time recovery — and in that case you already know where to find them.

**Don't touch two nodes at the same time.** With three nodes, quorum is sacred. One node at a time, no exceptions. If you stop two together, you're playing blindfolded Jenga.

**Don't start with the primary.** Always secondaries first, primary last. Doing it the other way round is the elegant way to invite chaos to dinner.

**Don't delete old binlogs immediately.** After the change, the old path `/mysql/bin_log/` won't be used for new files. But don't rush to `rm -rf /mysql/bin_log/*`. Wait. Verify that the cluster is stable, that new binlogs are being written to the new mount, that there are no errors in the MySQL log. Only after a few days of observation should you think about cleanup.

**Don't just trust the fact that "MySQL started".** MySQL can start but fail to rejoin the group. You need to verify three things: `log_bin_basename` points to the new path, the node is ONLINE in `replication_group_members`, and binlog files are actually being written in the new directory.

---

## What this operation really teaches

A filesystem at 92% isn't an emergency — it's a signal. The real problem wasn't disk space; it was an architectural choice made at installation time and never revisited: binlogs and data on the same volume.

Separating binary logs onto a dedicated volume isn't just a fix. It's infrastructure hardening. It's the difference between a system that "works" and one that's designed to keep working as things grow.

And the most important part of the entire intervention wasn't the `my.cnf` change — that's one line. The important part was the diagnosis: understanding what kind of cluster I was facing, checking the state of every node, preparing the storage, testing permissions, planning the execution order. All before touching a single parameter.

A senior DBA and a junior DBA both know the `systemctl stop mysqld` command. The difference is everything that happens before it.

------------------------------------------------------------------------

## Glossary

**[Group Replication](/en/glossary/group-replication/)** — MySQL's native mechanism for synchronous multi-node replication with automatic failover and quorum management. Supports single-primary and multi-primary modes.

**[Binary log](/en/glossary/binary-log/)** — MySQL's sequential binary record that tracks all data modifications (INSERT, UPDATE, DELETE, DDL), used for replication and point-in-time recovery.

**[GTID](/en/glossary/gtid/)** — Global Transaction Identifier — unique identifier assigned to every transaction in MySQL, simplifying replication management and transaction tracking across cluster nodes.

**[Quorum](/en/glossary/quorum/)** — Minimum number of nodes that must be active and communicating for a cluster to continue operating. In a 3-node cluster, quorum is 2.

**[Single-primary](/en/glossary/single-primary/)** — Group Replication mode where only one node accepts writes while the others are read-only with automatic failover.
