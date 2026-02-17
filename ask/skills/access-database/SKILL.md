---
name: access-database
allowed-tools: create_database
description: Connect to and query databases (PostgreSQL, MySQL, etc.). Use this whenever you need to check database connectivity, run queries, inspect schemas, or troubleshoot database issues. Use Python (not CLI tools like psql).
---
# Access Database

## ALWAYS
- Use Python with `psycopg2` (for PostgreSQL) — it is the most reliable approach.
- Search Kubernetes secrets for the connection string BEFORE connecting — databases often use non-standard ports.
- Chain `pip install` and the Python script in ONE execute call.
- Use the `create_database` tool to provision NEW databases (do not set up PostgreSQL manually).

## NEVER
- Do NOT assume port 5432 (PostgreSQL) or 3306 (MySQL) — production databases use non-standard ports.
- Do NOT try to install `psql` or `mysql` CLI tools — they may not be available and Python works better.
- Do NOT ask the user for database credentials — find them in Kubernetes secrets.
- Do NOT deploy PostgreSQL on Kubernetes yourself — use the `create_database` tool which provisions it on the NAS.

## Find database credentials in k8s secrets

```bash
# Search all secrets for database-related keys
kubectl get secrets -A -o json | grep -i 'database\|postgres\|mysql\|db_url\|dsn'

# Decode a specific secret
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

## Connect with Python (PostgreSQL)

```bash
pip install -q psycopg2-binary && python3 -c "
import psycopg2
conn = psycopg2.connect(
    host='192.168.1.2',
    port=<PORT>,        # GET THIS FROM K8S SECRETS - do NOT assume 5432
    dbname='<dbname>',
    user='<user>',
    password='<password>'
)
cur = conn.cursor()
cur.execute('SELECT 1')
print('Connection OK:', cur.fetchall())

# List tables
cur.execute(\"\"\"
    SELECT table_name FROM information_schema.tables
    WHERE table_schema = 'public'
\"\"\")
print('Tables:', [r[0] for r in cur.fetchall()])

conn.close()
"
```

## Connect with Python (MySQL)

```bash
pip install -q pymysql && python3 -c "
import pymysql
conn = pymysql.connect(
    host='<host>',
    port=<PORT>,
    user='<user>',
    password='<password>',
    database='<dbname>'
)
cur = conn.cursor()
cur.execute('SELECT 1')
print('Connection OK:', cur.fetchall())
conn.close()
"
```

## Parse connection strings
If you find a `DATABASE_URL` in secrets, parse it:
```
postgresql://user:pass@host:PORT/dbname
```
Extract the port — it is often NOT the default.

## Provision a new database
Use the `create_database` tool:
```
create_database('my-app')
```
This creates a PostgreSQL container on the NAS with auto-allocated port, credentials, and data directory.
