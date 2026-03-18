# 🧩 **Lab 12 – PostgreSQL Security, Access Control & Auditing (Hands-on Lab)**

---

## 🎯 **Objectives**

By the end of this lab, learners will be able to:

- Explain PostgreSQL **Role-Based Access Control (RBAC)**.
- Create and manage **roles and privileges** using `GRANT` and `REVOKE`.
- Configure **schema-level and object-level access** rules.
- Describe and configure **authentication methods** (`md5`, `scram-sha-256`).
- Implement **Row-Level Security (RLS)** policies.
- Enable **auditing and logging** for compliance (including `log_statement`, `log_connections`, `pgaudit`).
- Apply **encryption** concepts for data-at-rest (`pgcrypto` / disk) and data-in-transit (SSL/TLS).

---

## 🧭 **Introduction & Concept Overview**

PostgreSQL provides multiple layers of security:

- **Authentication** – who can connect (e.g., `md5`, `scram-sha-256`, `trust`, `peer`).
- **Authorization / RBAC** – what a role can do (`GRANT`, `REVOKE`, schema & table privileges).
- **Row-Level Security (RLS)** – which *rows* a role can see or modify.
- **Auditing & logging** – what activity is recorded.
- **Encryption** – how data is protected in storage and over the network.

In this lab, you will walk through each layer using a dedicated `securitylab` database.


| Topic              | Description                                                                   |
| ------------------ | ----------------------------------------------------------------------------- |
| **RBAC**           | Assign privileges based on roles instead of individual users                  |
| **GRANT / REVOKE** | Allow or remove access to schemas, tables, and specific operations            |
| **Authentication** | Control how passwords are verified (`md5`, `scram-sha-256`, etc.)             |
| **RLS**            | Restrict which rows a user can view or modify                                 |
| **Auditing**       | Record statements, logins, and activity via PostgreSQL settings or extensions |
| **Encryption**     | Protect data at rest (columns/disk) and in transit (SSL/TLS)                  |


---

## 🧰 **Step 0 – Setup**

Start with your running PostgreSQL instance or container (from previous labs):

```bash
docker exec -it postgres-container bash
psql -U postgres
```

Create a dedicated database for this lab:

```sql
CREATE DATABASE securitylab;
\c securitylab;
```

---

# 🧪 **Lab Exercises**

---

## 🔹 **Step 1 – Create Roles and Users (RBAC)**

### Goal

Define three key roles: `admin_role`, `dev_role`, and `readonly_role`.

```sql
-- 1️⃣ Create roles
CREATE ROLE admin_role NOINHERIT;
CREATE ROLE dev_role NOINHERIT;
CREATE ROLE readonly_role NOINHERIT;

-- 2️⃣ Create users and assign roles
CREATE USER admin_user WITH PASSWORD 'Admin@123';
CREATE USER dev_user WITH PASSWORD 'Dev@123';
CREATE USER readonly_user WITH PASSWORD 'Readonly@123';

-- 3️⃣ Grant roles to users
GRANT admin_role TO admin_user;
GRANT dev_role TO dev_user;
GRANT readonly_role TO readonly_user;
```

✅ **Observation:**
Each user now belongs to a role with specific privilege boundaries.

---

## 🔹 **Step 2 – Create a Schema and Base Table**

```sql
CREATE SCHEMA corp AUTHORIZATION admin_user;

CREATE TABLE corp.employee (
  emp_id SERIAL PRIMARY KEY,
  full_name TEXT,
  salary NUMERIC(10,2),
  dept TEXT
);

INSERT INTO corp.employee (full_name, salary, dept) VALUES
('Asha Singh', 1200000, 'IT'),
('Rohit Verma', 950000, 'Finance'),
('Pooja Das', 600000, 'HR');
```

---

## 🔹 **Step 3 – Grant / Revoke Privileges**

### Example Privilege Setup

```sql
-- Admin can do everything
GRANT ALL PRIVILEGES ON SCHEMA corp TO admin_role;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA corp TO admin_role;

-- Dev can read and write data
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA corp TO dev_role;

-- Readonly can only query
GRANT SELECT ON ALL TABLES IN SCHEMA corp TO readonly_role;

-- Revoke accidental access
REVOKE DELETE ON ALL TABLES IN SCHEMA corp FROM dev_role;
```

✅ **Observation:**
Privileges are assigned based on responsibilities — enforcing least privilege principle.

---

## 🔹 **Step 4 – Schema & Object Access Rules**

### Restrict schema visibility for non-admins

```sql
REVOKE USAGE ON SCHEMA corp FROM PUBLIC;
GRANT USAGE ON SCHEMA corp TO admin_role, dev_role, readonly_role;
```

✅ **Observation:**
Only assigned roles can access schema objects; others cannot even list the schema.

---

## 🔹 **Step 5 – Authentication Methods Overview**

> Authentication is configured in `**pg_hba.conf`**, located in the container’s `/var/lib/postgresql/data`.


| Method          | Description                           |
| --------------- | ------------------------------------- |
| `trust`         | No password (unsafe for production)   |
| `md5`           | MD5 password hashing                  |
| `scram-sha-256` | Modern, stronger password hashing     |
| `peer`          | System user match (local connections) |


### Check your current method:

```bash
cat /var/lib/postgresql/data/pg_hba.conf | grep -v '^#'
```

To switch to **SCRAM authentication**:

```bash
echo "host all all all scram-sha-256" >> /var/lib/postgresql/data/pg_hba.conf
```

Then restart PostgreSQL inside container:

```bash
pg_ctl -D /var/lib/postgresql/data restart
```

✅ **Observation:**
SCRAM is more secure and now used for password authentication.

---

## 🔹 **Step 6 – Row-Level Security (RLS)**

### Goal

Ensure each user can only view their own department’s data.

```sql
-- Enable RLS
ALTER TABLE corp.employee ENABLE ROW LEVEL SECURITY;

-- Policy for IT department
CREATE POLICY it_policy ON corp.employee
FOR SELECT USING (current_user = 'dev_user' AND dept = 'IT');

-- Policy for readonly_user (HR only)
CREATE POLICY hr_policy ON corp.employee
FOR SELECT USING (current_user = 'readonly_user' AND dept = 'HR');
```

Test as `dev_user`:

```sql
SET ROLE dev_user;
SELECT * FROM corp.employee;
RESET ROLE;
```

✅ **Observation:**
`dev_user` only sees IT employees; others are hidden.

---

## 🔹 **Step 7 – Auditing & Logging**

### 1️⃣ Enable Statement Logging (inside container)

Open `postgresql.conf`:

```bash
cat >> /var/lib/postgresql/data/postgresql.conf <<EOF
log_statement = 'all'
log_connections = on
log_disconnections = on
EOF
```

Restart service:

```bash
pg_ctl -D /var/lib/postgresql/data restart
```

✅ **Observation:**
Logs are written to `/var/lib/postgresql/data/log/` (or `/var/log/postgresql`).

---

### 2️⃣ Install and Use `pgaudit` (if available)

If the image supports it (Postgres 13+):

```sql
CREATE EXTENSION pgaudit;
ALTER SYSTEM SET pgaudit.log = 'all';
SELECT * FROM pg_available_extensions WHERE name='pgaudit';
```

✅ **Observation:**
Audits will now capture DDL/DML activity.

---

## 🔹 **Step 8 – Encryption (Data-at-Rest)**

### Using `pgcrypto` for column encryption

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Add encrypted column
ALTER TABLE corp.employee ADD COLUMN ssn BYTEA;

-- Encrypt sensitive data
UPDATE corp.employee
SET ssn = pgp_sym_encrypt('123-45-6789', 'SecretKey');

-- Decrypt to view
SELECT emp_id, full_name,
       convert_from(pgp_sym_decrypt(ssn, 'SecretKey'), 'UTF8') AS decrypted_ssn
FROM corp.employee;
```

✅ **Observation:**
SSNs stored as ciphertext in table; decrypted only when needed.

---

## 🔹 **Step 9 – Encryption (Data-in-Transit)**

### Enable SSL/TLS in PostgreSQL

Inside container:

```bash
cd /var/lib/postgresql/data
openssl req -new -x509 -days 365 -nodes \
  -text -out server.crt -keyout server.key -subj "/CN=postgres"
chmod 600 server.key
```

Edit `postgresql.conf`:

```bash
echo "ssl = on" >> /var/lib/postgresql/data/postgresql.conf
pg_ctl -D /var/lib/postgresql/data restart
```

From client:

```bash
psql "sslmode=require host=localhost dbname=securitylab user=postgres password=Admin@123"
```

✅ **Observation:**
Connection established securely over SSL.

---

## 🧾 **Summary**

- **RBAC & privileges**: use roles plus `GRANT`/`REVOKE` on schemas and tables to enforce least privilege.
- **Authentication**: configure secure password methods (prefer `scram-sha-256` over `md5`).
- **Row-Level Security (RLS)**: add per-row filters with `CREATE POLICY` on sensitive tables.
- **Auditing & logging**: use `log_statement`, `log_connections`, and optionally `pgaudit` for detailed activity records.
- **Encryption**: protect data at rest with `pgcrypto` (and/or disk encryption) and in transit with SSL/TLS.

---

## ✅ **Deliverables**

Submit **one `.sql` file** plus any required configuration snippets containing:

1. Role and user creation with `GRANT`/`REVOKE` for `admin_role`, `dev_role`, and `readonly_role`.
2. Schema and table setup in `securitylab` and example queries showing access differences between roles.
3. At least one **RLS policy** and a test query run under a restricted role.
4. `pgcrypto` encryption and decryption example for a sensitive column.
5. A short note or screenshot confirming **logging/auditing** and **SSL/TLS** settings.

---

## 🧩 **Practice Challenges**

1. Create a custom RLS policy where a manager can see all employees in their department, but regular users only see themselves.
2. Revoke all write permissions from `readonly_user` and verify enforcement by attempting `INSERT`/`UPDATE`/`DELETE`.
3. Encrypt multiple fields (like `salary`, `email`) using different keys and document how key rotation would work.
4. Enable `log_duration` or `log_min_duration_statement` and analyze query execution times from the logs.
5. Use an external SSL certificate (self-signed CA + server cert) to replace the simple lab certificate.

---

## 🧹 **Cleanup**

When you are done with this lab, you can remove the lab database and roles:

```sql
DROP DATABASE IF EXISTS securitylab;

REVOKE admin_role FROM admin_user;
REVOKE dev_role FROM dev_user;
REVOKE readonly_role FROM readonly_user;

DROP ROLE IF EXISTS admin_role;
DROP ROLE IF EXISTS dev_role;
DROP ROLE IF EXISTS readonly_role;

DROP USER IF EXISTS admin_user;
DROP USER IF EXISTS dev_user;
DROP USER IF EXISTS readonly_user;
```

