# 🧩 **Lab 14 – PostgreSQL Backup, Recovery & High Availability (Hands-on Lab)**

---

## 🧭 **Introduction**

Backups and high availability are not optional in production PostgreSQL. This lab teaches practical, operator-grade basics:

- **Logical backups** for portability and selective restores (`pg_dump`, `pg_dumpall`, `pg_restore`)
- **Physical backups** and how they relate to **WAL** and **PITR**
- **Streaming replication** concepts and a demo setup
- HA tooling patterns (pgPool-II / Patroni) and scaling/failover strategies

---

## 🎯 **Objectives**

By the end of this lab, learners will:

* Understand different **backup types** – logical vs physical
* Use **pg_dump**, **pg_dumpall**, and **pg_restore**
* Perform **Point-in-Time Recovery (PITR)**
* Explore the **Write-Ahead Log (WAL)** concept
* Set up basic **streaming replication** (demo-level)
* Learn about **pgPool-II / Patroni** for HA management
* Simulate **failover and scaling strategies**

---

## 🧠 **Concept Overview**

| Topic                             | Description                                                                           |
| --------------------------------- | ------------------------------------------------------------------------------------- |
| **Logical Backup**                | Uses SQL-based export tools (`pg_dump`, `pg_dumpall`); portable and easy to restore.  |
| **Physical Backup**               | Copies actual data directory files; suitable for large databases and PITR.            |
| **WAL (Write-Ahead Log)**         | Records every change before committing; essential for crash recovery and replication. |
| **PITR (Point-in-Time Recovery)** | Allows restoring database to a specific timestamp using WAL logs.                     |
| **Streaming Replication**         | Replica continuously fetches WAL segments from primary.                               |
| **pgPool-II / Patroni**           | Middleware tools for load balancing, connection pooling, and automatic failover.      |

---

## 🧰 **Setup**

Start or reuse your containerized PostgreSQL instance:

```bash
docker run -d --name pg-backup \
  -e POSTGRES_PASSWORD=Admin@123 \
  -v /tmp/pgdata:/var/lib/postgresql/data \
  postgres:15

docker exec -it pg-backup bash
psql -U postgres
```

---

## 🧰 **Step 0 – Setup (make this lab independent)**

Create a demo database:

```sql
CREATE DATABASE backup_lab;
\c backup_lab;
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  full_name TEXT,
  dept TEXT,
  salary NUMERIC(10,2)
);
INSERT INTO employees (full_name, dept, salary)
VALUES ('Asha Singh','IT',1200000),
       ('Rohit Verma','Finance',950000),
       ('Pooja Das','HR',600000);
```

---

# 🧪 **Lab Exercises**

---

## 🔹 **Step 1 – Logical Backup using `pg_dump`**

### Single Database Backup

```bash
pg_dump -U postgres -d backup_lab -F c -f /tmp/backup_lab.dump
```

✅ **Observation:**
A `.dump` file is created with compressed format (`-F c`).

### Useful variations (optional)

```bash
# Schema-only
pg_dump -U postgres -d backup_lab --schema-only -f /tmp/backup_lab_schema.sql

# Data-only
pg_dump -U postgres -d backup_lab --data-only -f /tmp/backup_lab_data.sql
```

---

### Restore the Dump

```bash
createdb restore_lab -U postgres
pg_restore -U postgres -d restore_lab /tmp/backup_lab.dump
```

Verify:

```sql
\c restore_lab
SELECT * FROM employees;
```

✅ **Observation:**
Data restored successfully into the new database.

---

## 🔹 **Step 2 – Full Cluster Backup using `pg_dumpall`**

```bash
pg_dumpall -U postgres > /tmp/full_backup.sql
```

Restore:

```bash
psql -U postgres -f /tmp/full_backup.sql
```

✅ **Observation:**
`pg_dumpall` exports all databases, roles, and global objects.

---

## 🔹 **Step 3 – WAL Basics (Write-Ahead Log)**

WAL records changes before they are written to data files. It powers crash recovery, replication, and PITR.

Run inside `psql`:

```sql
SELECT pg_current_wal_lsn() AS current_wal_lsn;
SELECT pg_walfile_name(pg_current_wal_lsn()) AS current_wal_segment;
```

✅ **Observation:**
You can see the current WAL position and the WAL segment name PostgreSQL is writing.

---

## 🔹 **Step 3 – Simulate Accidental Delete and Recovery**

```sql
\c backup_lab
DELETE FROM employees WHERE dept='Finance';
SELECT * FROM employees;
```

### Restore from Backup

```bash
dropdb backup_lab -U postgres
createdb backup_lab -U postgres
pg_restore -U postgres -d backup_lab /tmp/backup_lab.dump
```

✅ **Observation:**
Deleted data recovered from last backup.

---

## 🔹 **Step 4 – Physical Backup (Base + WAL)**

### Stop the PostgreSQL service inside container

```bash
pg_ctl -D /var/lib/postgresql/data stop
```

### Copy Data Directory

```bash
cp -r /var/lib/postgresql/data /var/lib/postgresql/data_bkp
```

Start PostgreSQL again:

```bash
pg_ctl -D /var/lib/postgresql/data start
```

✅ **Observation:**
This backup contains full binary data including WAL logs — suitable for PITR.

---

## 🔹 **Step 5 – Point-in-Time Recovery (PITR)**

### Enable WAL Archiving

Edit configuration:

```bash
mkdir -p /var/lib/postgresql/wal_archive
echo "archive_mode = on" >> /var/lib/postgresql/data/postgresql.conf
echo "archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'" >> /var/lib/postgresql/data/postgresql.conf
pg_ctl -D /var/lib/postgresql/data restart
```

### Insert Extra Data and Note Time

```sql
INSERT INTO employees VALUES (4, 'Deleted User', 'IT', 1000000);
SELECT now();  -- Note the timestamp
```

Assume accidental delete:

```sql
DELETE FROM employees;
```

### Restore to Earlier Time

1. Stop PostgreSQL:

   ```bash
   pg_ctl -D /var/lib/postgresql/data stop
   ```

2. Remove old data directory and restore base backup:

   ```bash
   rm -rf /var/lib/postgresql/data
   cp -r /var/lib/postgresql/data_bkp /var/lib/postgresql/data
   ```

3. Create recovery file:

   ```bash
   echo "restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'" >> /var/lib/postgresql/data/postgresql.conf
   echo "recovery_target_time = '2025-11-05 10:30:00+05:30'" >> /var/lib/postgresql/data/postgresql.conf
   touch /var/lib/postgresql/data/recovery.signal
   ```

4. Start PostgreSQL:

   ```bash
   pg_ctl -D /var/lib/postgresql/data start
   ```

✅ **Observation:**
Database rolls back to the timestamp **before deletion**.

---

## 🔹 **Step 6 – Streaming Replication (Demo Level)**

### 1️⃣ Create Primary and Replica Containers

```bash
docker run -d --name pg-primary -e POSTGRES_PASSWORD=Admin@123 postgres:15
docker run -d --name pg-replica -e POSTGRES_PASSWORD=Admin@123 postgres:15
```

### 2️⃣ Configure Primary

```bash
docker exec -it pg-primary bash
echo "wal_level = replica" >> /var/lib/postgresql/data/postgresql.conf
echo "max_wal_senders = 3" >> /var/lib/postgresql/data/postgresql.conf
echo "listen_addresses = '*'" >> /var/lib/postgresql/data/postgresql.conf
echo "host replication all 0.0.0.0/0 trust" >> /var/lib/postgresql/data/pg_hba.conf
pg_ctl -D /var/lib/postgresql/data restart
```

Create a replication user (recommended for real setups; demo uses trust above):

```bash
docker exec -it pg-primary psql -U postgres -c "CREATE ROLE repl WITH REPLICATION LOGIN PASSWORD 'Repl@123';"
```

Create a demo database/table on the primary:

```bash
docker exec -it pg-primary psql -U postgres -c "CREATE DATABASE repl_lab;"
docker exec -it pg-primary psql -U postgres -d repl_lab -c "CREATE TABLE IF NOT EXISTS employees(id int primary key, full_name text, dept text, salary numeric(10,2));"
docker exec -it pg-primary psql -U postgres -d repl_lab -c "INSERT INTO employees VALUES (1,'ReplicaSeed','IT',700000) ON CONFLICT (id) DO NOTHING;"
```

### 3️⃣ Take Base Backup for Replica

```bash
docker exec -it pg-replica bash -lc "rm -rf /var/lib/postgresql/data/* && pg_basebackup -h pg-primary -D /var/lib/postgresql/data -U repl -Fp -Xs -P -R"
```

✅ **Observation:**
Replica now continuously applies WAL files from the primary server.

Test by inserting a new row in primary:

```sql
\c repl_lab
INSERT INTO employees VALUES (2, 'ReplicaTest', 'IT', 700000);
```

Check replica:

```sql
\c repl_lab
SELECT * FROM employees;
```

✅ **Result:**
Row appears automatically — replication successful.

---

## 🔹 **Step 7 – Overview of pgPool-II & Patroni**

| Tool          | Function                                                                                                     |
| ------------- | ------------------------------------------------------------------------------------------------------------ |
| **pgPool-II** | Middleware for connection pooling, load balancing, and replication management.                               |
| **Patroni**   | Manages PostgreSQL clusters with automatic failover and distributed consensus (usually with etcd or Consul). |

> Demo-level Note: Installing full Patroni/pgPool requires multiple containers and a distributed setup; use for conceptual demonstration only.

---

## 🔹 **Step 8 – Scaling & Failover Strategies**

| Scenario          | Strategy                                    |
| ----------------- | ------------------------------------------- |
| Read Scaling      | Add read replicas via streaming replication |
| Write Scaling     | Use sharding or partitioned design          |
| Failover          | Use pgPool-II or Patroni to promote standby |
| Backup Retention  | Automate daily `pg_dump` + WAL archiving    |
| Disaster Recovery | Replicate backups to remote region/cloud    |

---

## 🧾 **Summary Table**

| Area                  | Key Tools / Commands                         |
| --------------------- | -------------------------------------------- |
| Logical Backup        | `pg_dump`, `pg_restore`, `pg_dumpall`        |
| Physical Backup       | Copy data directory or use `pg_basebackup`   |
| PITR                  | WAL + `recovery_target_time`                 |
| WAL                   | `archive_mode`, `archive_command`            |
| Streaming Replication | `wal_level=replica`, `pg_basebackup`         |
| HA Tools              | pgPool-II, Patroni                           |
| Failover / Scaling    | Read replicas, load balancing, auto-failover |

---

## ✅ **Deliverables**

Each learner should:

1. Capture screenshot of backup and restore process
2. Demonstrate PITR recovery before accidental deletion
3. Show replication output (row visible in replica)
4. Submit short note explaining WAL and failover process

---

## 🧩 **Practice Challenges**

1. Automate daily logical backups using a cron job inside the container.
2. Simulate replication delay using `pg_wal_replay_pause()` and resume with `pg_wal_replay_resume()`.
3. Test how WAL archives grow over time and analyze retention strategy.
4. Research how **Patroni** achieves automatic leader election using `etcd`.
5. Configure **pgPool-II** load balancing between primary and replica for read queries.

---

## 🧹 **Cleanup (optional, to avoid conflicts with future labs)**

When you are done:

```sql
-- From psql
DROP DATABASE IF EXISTS backup_lab;
DROP DATABASE IF EXISTS restore_lab;
DROP DATABASE IF EXISTS repl_lab;
```

```bash
# Stop and remove demo containers (optional)
docker rm -f pg-backup pg-primary pg-replica
```
