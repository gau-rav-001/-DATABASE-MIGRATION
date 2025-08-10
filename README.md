# -DATABASE-MIGRATION

*COMPANY NAME* : CODETECH IT SOLUTIONS

*NAME*: GAURAV kUMBHARE

*INTERN ID*: CT04DH2634

*DOMAIN*: SQL

*DURATION*: 4 WEEKS

*MENTOR*: NEELA SANTOSH KUMAR

*DISCRIPTION*

MySQL → PostgreSQL Migration: Deliverables

Project: Migrate data from MySQL to PostgreSQL and ensure data integrity

Deliverables included in this package

migration/ directory (logical):

01_dump_mysql.sh — script to export MySQL schema & data

02_convert_schema.sql — schema conversion SQL (examples and fixes)

03_pgloader.load — pgloader config to migrate data and transform types

04_data_validate.sql — verification & checksum queries for integrity checks

05_post_migration_tasks.sql — indexes, sequences, constraints, and cleanup

06_rollback_instructions.md — how to revert changes

07_python_etl.py — Python ETL for complex rows (optional)

README.md — how to run scripts, prerequisites, and order of operations

summary_report.md — human-readable migration report containing scope, assumptions, steps executed, test results, issues found, and recommendations.

Quick Executive Summary (for summary_report.md)

Goal: Move database shopdb from MySQL (source) to PostgreSQL (target) while preserving data accuracy, referential integrity, and acceptable downtime.

High-level approach:

Export MySQL schema and data (logical export using mysqldump).

Convert schema differences (data types, auto-increment → sequences, ENUMs, charset/collation).

Use pgloader for bulk data transfer where possible; use targeted Python ETL for complex transformations.

Recreate constraints, indexes, sequences in PostgreSQL after data load.

Validate data using row counts, checksums, and random record comparison.

Run application tests and performance checks.

Deliverables: Migration scripts (see above), verification queries, rollback plan, and the summary report.

Prerequisites (included in README.md)

Access & credentials for MySQL and PostgreSQL servers with necessary privileges.

Tools installed on migration host:

mysqldump (MySQL client)

pg_restore / psql (Postgres client)

pgloader (recommended)

python3 with mysql-connector-python and psycopg2-binary (if using Python ETL)

Sufficient disk space for dumps.

Maintenance window & backups scheduled.

Files: Detailed contents and sample code

01_dump_mysql.sh

#!/usr/bin/env bash
set -euo pipefail
# Usage: ./01_dump_mysql.sh shopdb
DB_NAME=${1:-shopdb}
OUT_DIR=./migration/dumps
mkdir -p "$OUT_DIR"
# dump schema
mysqldump --single-transaction --no-data --routines --events -u root -p"1234" "$DB_NAME" > "$OUT_DIR/${DB_NAME}_schema.sql"
# dump data
mysqldump --single-transaction --no-create-info --skip-triggers -u root -p"1234" "$DB_NAME" > "$OUT_DIR/${DB_NAME}_data.sql"
# dump triggers separately
mysqldump --routines --no-create-db --no-data -u root -p"1234" "$DB_NAME" > "$OUT_DIR/${DB_NAME}_routines.sql"

echo "Dumps created in $OUT_DIR"

Note: Never hardcode production credentials; use environment variables or a secure secrets manager.

02_convert_schema.sql (example snippets)

Replace MySQL TINYINT(1) used as boolean → BOOLEAN in Postgres

Convert AUTO_INCREMENT columns: create sequence and set DEFAULT nextval('seq')

Convert ENUM types → VARCHAR or create CREATE TYPE in Postgres

Example transform for an orders table:

-- MySQL auto-increment column mapped to Postgres sequence
CREATE SEQUENCE orders_id_seq;
CREATE TABLE orders (
    id integer PRIMARY KEY DEFAULT nextval('orders_id_seq'),
    customer_id integer NOT NULL,
    total numeric(12,2),
    status varchar(20),
    created_at timestamp without time zone
);
-- After load, set sequence to max(id)
SELECT setval('orders_id_seq', (SELECT COALESCE(MAX(id),0) FROM orders));

03_pgloader.load (example)

LOAD DATABASE
     FROM mysql://root:1234@localhost/shopdb
     INTO postgresql://postgres:admin@localhost/shopdb_pg

 WITH include drop, create tables, create indexes, reset sequences
 SET maintenance_work_mem to '512MB', work_mem to '64MB'
 CAST type tinyint to boolean using tinyint-to-boolean,
      type datetime to timestamptz USING zero-datetime-to-null

 BEFORE LOAD DO
   $$ create schema if not exists public; $$;

 AFTER LOAD DO
   $$ SELECT setval(pg_get_serial_sequence('orders','id'), coalesce(max(id),1), true) FROM orders; $$;

Notes: tune casting rules for your schema; pgloader supports many automatic casts.

04_data_validate.sql

Row counts per table

Checksums (MD5 of concatenated columns) per table or partition

Referential checks for orphaned FKs

Example queries:

-- Row count
SELECT 'orders' as table, COUNT(*) FROM public.orders;

-- Simple checksum for table (works for reasonably sized tables)
SELECT md5(string_agg(t.row_text, '||')) as checksum FROM (
  SELECT md5(COALESCE(id::text,'') || '|' || COALESCE(customer_id::text,'') || '|' || COALESCE(total::text,'')) as row_text
  FROM public.orders
  ORDER BY id
) t;

-- Referential integrity: find orders with missing customers
SELECT o.id FROM public.orders o LEFT JOIN public.customers c ON o.customer_id = c.id WHERE c.id IS NULL LIMIT 10;

Validation process:

Compare row counts (MySQL vs Postgres) for each table.

Compare checksums (per-table) or sample rows by primary key.

Run application test suite.

Validate indexes and query plans for critical queries.

05_post_migration_tasks.sql

Recreate foreign key constraints if you skipped them for load speed

Create or re-create indexes

Add triggers, stored procedures (translated to PL/pgSQL)

Adjust ownership and grants

Example:

ALTER TABLE public.orders ADD CONSTRAINT fk_orders_customers FOREIGN KEY (customer_id) REFERENCES public.customers(id);
CREATE INDEX idx_orders_customer_id ON public.orders(customer_id);
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;

06_rollback_instructions.md

If using new Postgres database copy: DROP the target DB and restore from pre-migration backup.

If doing in-place migration, have backups of pre-change schema and data. Steps:

Stop application.

Restore MySQL from mysqldump (mysql < dump.sql) or from binary backup.

Repoint application to MySQL.

Safety tip: Always run first migrations to a staging environment and test rollback there before touching production.

07_python_etl.py (optional — for complex transformations)

#!/usr/bin/env python3
# Purpose: transform and copy rows from MySQL to Postgres when pgloader not sufficient
import mysql.connector
import psycopg2
from psycopg2.extras import execute_values

MYSQL_CFG = { 'host':'localhost','user':'root','password':'1234','database':'shopdb' }
PG_CFG = "dbname='shopdb_pg' user='postgres' host='localhost' password='admin'"
BATCH = 1000

def fetch_mysql_rows(cursor, last_id=0):
    cursor.execute('SELECT id, customer_id, total, status, created_at FROM orders WHERE id > %s ORDER BY id LIMIT %s', (last_id, BATCH))
    return cursor.fetchall()

with mysql.connector.connect(**MYSQL_CFG) as mconn, psycopg2.connect(PG_CFG) as pconn:
    mcur = mconn.cursor(dictionary=True)
    pcur = pconn.cursor()
    last = 0
    while True:
        rows = fetch_mysql_rows(mcur, last)
        if not rows:
            break
        values = [(r['id'], r['customer_id'], r['total'], r['status'], r['created_at']) for r in rows]
        execute_values(pcur, "INSERT INTO public.orders (id, customer_id, total, status, created_at) VALUES %s ON CONFLICT (id) DO UPDATE SET customer_id = EXCLUDED.customer_id, total = EXCLUDED.total, status=EXCLUDED.status, created_at=EXCLUDED.created_at", values)
        pconn.commit()
        last = rows[-1]['id']

Testing & Verification Plan (to include in summary_report.md)

Pre-migration checks: schema differences report, identify unsupported features (e.g., MySQL-specific stored procs, user-defined functions, etc.).

Smoke test on staging: migrate a copy of production and run full app test suite.

Data integrity checks: row counts, checksums, FK checks, nullability checks.

Functional tests: run app-level integration tests.

Performance tests: run key queries and compare explain plans; tune indexes and configuration.

Sign-off checklist: stakeholders approve after validation.

Risks & Mitigations

Risk: Loss of precision for numeric/date types. Mitigation: Map MySQL DATETIME → TIMESTAMP with timezone if required; review decimal precisions.

Risk: Stored procedures and triggers incompatible. Mitigation: Translate to PL/pgSQL and test thoroughly.

Risk: Collation/charset differences leading to different sort orders. Mitigation: Ensure correct ENCODING and LC_COLLATE settings when creating the Postgres DB.

Rollout Plan (for production cutover)

Prepare and test in staging.

Schedule maintenance window.

Lock writes (read-only mode) or put application in maintenance.

Final incremental export (binlog-based / incremental sync) to capture recent changes.

Run pgloader/ETL for final sync.

Switch application to PostgreSQL and run smoke tests.

Monitor and rollback if critical issues.

What I am delivering here

A ready-to-run set of example scripts (bash + pgloader + SQL + optional Python) to migrate data.

A template summary_report.md (this document) to fill with environment-specific details, timestamps, and logs.

A verification script set and instructions for rollback.

Next steps for you (action items)

Fill in production credentials using secure env vars.

Run migration on staging and execute the validation queries in 04_data_validate.sql.

Review stored procedures/triggers and translate those needed to Postgres.

Share logs and failed-record samples if any rows fail validation.

