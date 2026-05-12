# Hevo Data Engineering Assignment
### Subhankar Kar Chowdhury


---


## The Objective

This assignment required building a complete, production-grade end-to-end ELT pipeline,
from a locally hosted PostgreSQL database all the way through to a transformed, tested,
and documented data model in Snowflake. 

The specific deliverables were:

1. A Docker-based PostgreSQL database loaded with three raw datasets
2. A Hevo pipeline using Logical Replication to replicate data into Snowflake in near real time
3. A dbt project that transforms raw data into a clean, tested customers mart table
4. A GitHub repository with no hardcoded credentials, a working README, and a Loom walkthrough


---


## Architecture

```
+------------------+    Logical       +--------+    COPY    +-----------+    dbt     +---------------+
|  PostgreSQL 15   |  Replication     |  Hevo  |  ------->  | Snowflake |  ------->  |   customers   |
|  (Docker-based,  |  ------------->  |  ELT   |            | Warehouse |   models   |  (mart table) |
|  local machine)  |  WAL streaming   |        |            |           |            |               |
+------------------+                  +--------+            +-----------+            +---------------+
   raw_customers                                            HEVO_SF_RAW_CUSTOMERS    - customer_id
   raw_orders                                               HEVO_SF_RAW_ORDERS       - first_name
   raw_payments                                             HEVO_SF_RAW_PAYMENTS     - last_name
                                                                                     - first_order
                                                                                     - most_recent_order
                                                                                     - number_of_orders
                                                                                     - customer_lifetime_value
```

Each layer has a single, clear responsibility:

- **PostgreSQL** owns the raw source data
- **Hevo** owns the movement of data (no transformation)
- **Snowflake** owns the storage
- **dbt** owns all transformation logic

This separation means any layer can be replaced or upgraded without breaking the others.


---


## Components — The Role of Each

### PostgreSQL (Source Database)
PostgreSQL is one of the most widely used open-source relational databases in enterprise
environments. It supports Logical Replication natively — essential for real-time CDC
(Change Data Capture). Running it inside Docker means zero installation complexity
and complete reproducibility across machines.

### Docker
Docker runs PostgreSQL inside a self-contained container on the local machine —
no complex database installation required, the same environment runs on any machine,
and it is easy to start, stop, and reset for testing.

### ngrok
Hevo is a cloud service. A local Docker database has no public internet address.
ngrok creates a secure TCP tunnel — a temporary public address pointing to the local
database — allowing Hevo Cloud to connect to it as if it were hosted on a public server.
This is a real networking challenge that arises whenever Hevo customers have on-premise
or locally hosted databases.

### Hevo Data (ELT Pipeline)
Hevo moves data from PostgreSQL to Snowflake automatically, in near real time,
using Logical Replication. This means:

- Every INSERT, UPDATE, and DELETE is captured as it happens
- Sub-minute latency from source change to Snowflake landing
- No scheduled batch jobs to manage or monitor
- Always-fresh data for downstream analytics

### Snowflake (Cloud Data Warehouse)
Snowflake is the destination warehouse where all raw data lands and where dbt runs
transformations. It was provisioned via Snowflake Partner Connect — which
auto-creates the database, warehouse, role, and user for Hevo, removing all
manual configuration steps.

### dbt (Data Transformation Layer)
dbt transforms three raw, scattered tables into one clean, business-ready customers
table. Unlike writing SQL directly in Snowflake, dbt adds:

- **Version control** — every transformation is a tracked file in Git
- **Automated testing** — data quality is checked on every single run
- **Auto-generated documentation** — every column described and lineaged
- **Modular SQL** — reusable building blocks with no duplicated logic


---


## Prerequisites

| Tool | Version Used | Purpose |
|------|-------------|---------|
| Docker Desktop | postgres:15 image | Runs PostgreSQL locally in a container |
| Python | 3.11.15 | Required runtime for dbt |
| dbt-snowflake | 1.11.9 (core) / 1.11.4 (adapter) | dbt with Snowflake connector |
| ngrok | Latest free tier | Exposes local database to Hevo Cloud |
| Snowflake Account | Trial (30 days) | Cloud data warehouse destination |
| Hevo Account | Trial via Snowflake Partner Connect | ELT pipeline tool |
| Git | Any recent version | Version control and GitHub push |



---


## Step-by-Step Implementation Guide

### Step 1: Install Required Tools

Install Docker Desktop from https://docker.com and Git from https://git-scm.com.
Then install Python 3.11 and dbt via Homebrew:

```bash
brew install python@3.11
python3.11 --version                    # Should show Python 3.11.x
```

Create a Python virtual environment specifically for dbt. This isolates dbt's
dependencies from the rest of your system and prevents version conflicts:

```bash
cd ~/Desktop
python3.11 -m venv dbt-env
source dbt-env/bin/activate             # Activate before every dbt session
pip install --upgrade pip
pip install dbt-snowflake
dbt --version                           # Should show dbt Core 1.x.x
```

Install ngrok from https://ngrok.com, create a free account, and connect it:

```bash
brew install ngrok
ngrok config add-authtoken YOUR_TOKEN_HERE
```

---

### Step 2: Start PostgreSQL in Docker

Docker runs PostgreSQL in a self-contained container. The `-p 5432:5432` flag
maps the container's database port to your local machine so tools like DBeaver
can connect to it:


```
Set these in the terminal before running:

export POSTGRES_USER="your_db_username"
export POSTGRES_PASSWORD="your_db_password"
export POSTGRES_DB="your_db_name"
```

```bash
docker run --name hevo-postgres \
  -e POSTGRES_USER=$POSTGRES_USER \
  -e POSTGRES_PASSWORD=$POSTGRES_PASSWORD \
  -e POSTGRES_DB=$POSTGRES_DB \
  -p 5432:5432 \
  -d postgres:15
```

Verify it is running:

```bash
docker ps
```

You should see `hevo-postgres` listed with status `Up`.

---

### Step 3: Create Tables and Load CSV Data

Connect to the database using DBeaver (Host: localhost, Port: 5432, Database: your_db_name, User: your_db_username, Password: your_db_password)

Open a SQL Editor and create the three tables:

```sql
CREATE TABLE raw_customers (
    id INTEGER PRIMARY KEY,
    first_name VARCHAR(50),
    last_name  VARCHAR(50)
);

CREATE TABLE raw_orders (
    id         INTEGER PRIMARY KEY,
    user_id    INTEGER,
    order_date DATE,
    status     VARCHAR(50)
);

CREATE TABLE raw_payments (
    id             INTEGER PRIMARY KEY,
    order_id       INTEGER,
    payment_method VARCHAR(50),
    amount         INTEGER
);
```

Import each CSV file using DBeaver's Import Data feature:
right-click each table → Import Data → CSV → select the file → Finish.

Verify the row counts:

```sql
SELECT COUNT(*) FROM raw_customers;   -- Expected: 100
SELECT COUNT(*) FROM raw_orders;      -- Expected: 99
SELECT COUNT(*) FROM raw_payments;    -- Expected: 113
```

---

### Step 4: Enable Logical Replication in PostgreSQL

This is the most critical configuration step. Logical Replication must be
enabled on the PostgreSQL side — Hevo cannot do this itself. It tells the
database to keep a detailed change log (WAL — Write Ahead Log) that Hevo
can subscribe to for real-time streaming.

Go inside the Docker container:

```bash
docker exec -it hevo-postgres psql -U $POSTGRES_USER -d $POSTGRES_DB
```

Run these three settings:

```sql
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;
```

Exit and restart Docker for the settings to take effect:

```bash
\q
docker restart hevo-postgres
```

Re-enter and verify the settings applied:

```bash
docker exec -it hevo-postgres psql -U $POSTGRES_USER -d $POSTGRES_DB
SHOW wal_level;                         -- Must show: logical
```

Grant the replication role to the database user:

```sql
ALTER ROLE your_db_username REPLICATION LOGIN;
```

Create a named publication — this tells PostgreSQL exactly which tables
Hevo is allowed to replicate. Only these three tables will be streamed:

```sql
CREATE PUBLICATION hevo_publication
FOR TABLE raw_customers, raw_orders, raw_payments;
```

Verify the publication was created:

```sql
\dRp+
```

You should see `hevo_publication` listing all three tables. Type `\q` to exit.

---

### Step 5: Create the ngrok Tunnel

Hevo is a cloud service and cannot reach a database running on a local laptop.
ngrok solves this by creating a secure public-facing TCP tunnel that points
directly to the local PostgreSQL port.

Open a new Terminal window and run:

```bash
ngrok tcp 5432
```

Keep this Terminal window open for the entire duration of the Hevo pipeline.
Note the Forwarding address — it looks like:

```
Forwarding: tcp://X.tcp.ngrok.io:XXXXX -> localhost:5432
```

The host (`X.tcp.ngrok.io`) and port (`XXXXX`) will be provided
to Hevo as the database connection details. ngrok generates a new address
every time it restarts, so if it stops, the Hevo pipeline source must be
updated with the new address.

---

### Step 6: Sign Up for Snowflake

Go to https://signup.snowflake.com and create a free 30-day trial account.
Select AWS as the cloud provider and choose the region closest - 
Mumbai was chosen


---

### Step 7: Sign Up for Hevo via Snowflake Partner Connect

The assignment specifically requires signing up through Snowflake, not directly.
This automatically provisions the Snowflake destination with correct permissions.

1. Inside Snowflake, go to **Admin → Partner Connect**
2. Find **Hevo Data** and click **Connect**
3. Snowflake auto-creates `PC_HEVODATA_DB`, `COMPUTE_WH`, role, and user
4. You will be redirected to Hevo to complete signup
5. Note your **Team ID** from Hevo Settings (also visible in the URL: `/team/XXXXXX/`)

---

### Step 8: Build the Hevo Pipeline

In Hevo, go to **Pipelines → Create Pipeline → PostgreSQL**.

Configure the source using the ngrok tunnel details:

- **Host:** ngrok forwarding host (e.g. `6.tcp.ngrok.io`)
- **Port:** ngrok forwarding port (e.g. `15432`)
- **Database:** your_db_name
- **User:** your_db_username
- **Ingestion Mode:** `Logical Replication` — required by the assignment
- **Publication Name:** `hevo_publication`

Set the destination to the Snowflake account provisioned by Partner Connect.
Click **Test Connection** — it should pass. Start the pipeline.

Note your **Pipeline ID** from the URL: `/pipelines/XXXXXX/`

Wait for the Historical Load to complete. Verify in Snowflake:

```sql
SELECT COUNT(*) FROM PC_HEVODATA_DB.PUBLIC.HEVO_SF_RAW_CUSTOMERS;  -- 100
SELECT COUNT(*) FROM PC_HEVODATA_DB.PUBLIC.HEVO_SF_RAW_ORDERS;     -- 99
SELECT COUNT(*) FROM PC_HEVODATA_DB.PUBLIC.HEVO_SF_RAW_PAYMENTS;   -- 113
```

---

### Step 9: Configure dbt

Set your Snowflake credentials as environment variables. These must be set
in every new Terminal session before running dbt — they are never stored
in any project file:

```bash
export SNOWFLAKE_ACCOUNT="your_orgname-accountname"
export SNOWFLAKE_USER="your_snowflake_username"
export SNOWFLAKE_PASSWORD="your_snowflake_password"
```

Create the dbt profile file outside the project folder so it is never
accidentally committed to GitHub:

```bash
mkdir -p ~/.dbt
```

Create `~/.dbt/profiles.yml` with this content:

```yaml
hevo_subhankar_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: ACCOUNTADMIN
      database: PC_HEVODATA_DB
      warehouse: COMPUTE_WH
      schema: PUBLIC
      threads: 1
```

Activate the virtual environment and verify the connection:

```bash
source ~/Desktop/dbt-env/bin/activate
cd ~/Desktop/hevo_subhankar_project
dbt debug
```

All checks should show `OK`.

---

### Step 10: Run dbt and Test

Build the customers model:

```bash
dbt run
```

dbt executes the SQL in `customers.sql`, joins the three source tables,
calculates all required metrics, and materialises the result as a permanent
table in Snowflake at `PC_HEVODATA_DB.PUBLIC.CUSTOMERS`.

Run data quality tests:

```bash
dbt test
```

dbt runs all tests defined in `schema.yml`. If any test fails, dbt reports
exactly which rows failed and why.

**Expected result:** `PASS=6, WARN=0, ERROR=0`

Verify the final output in Snowflake:

```sql
SELECT * FROM PC_HEVODATA_DB.PUBLIC.CUSTOMERS
ORDER BY customer_lifetime_value DESC
LIMIT 10;
```

---

### Step 11: Push to GitHub

```bash
git add .
git commit -m "Complete dbt project with customers model and tests"
git push origin main
```

Verify no credentials are present anywhere in the repository:

```bash
git grep -r "password" --include="*.yml" --include="*.sql"
```

Only `env_var` references should appear — never literal values.

---

## Project Structure

```
Hevo-assignment_Subhankar/
│
├── models/
│   │
│   ├── sources.yml
│   │     Declares the source tables Hevo loaded into Snowflake.
│   │     Tells dbt which database, schema, and table names to
│   │     read from. Without this, dbt cannot find the raw data.
│   │
│   ├── customers.sql
│   │     The core transformation. Joins raw_customers, raw_orders,
│   │     and raw_payments. Calculates first_order, most_recent_order,
│   │     number_of_orders, and customer_lifetime_value. Uses COALESCE
│   │     to replace NULL with 0 for customers with no orders yet.
│   │
│   └── schema.yml
│         Defines automated data quality tests that run after every
│         dbt build: unique and not_null on customer_id, not_null on
│         first_name, number_of_orders, and customer_lifetime_value.
│
├── dbt_project.yml
│     Project configuration — name, version, model paths,
│     and materialisation settings (table vs view).
│
├── .gitignore
│     Excludes credentials, dbt build artifacts, and Mac
│     system files (.DS_Store) from version control.
│
└── README.md
      This document.
```

---

## Final Output — The Customers Table

The customers mart table contains one row per customer:

| Column | Source Table | Description |
|--------|-------------|-------------|
| customer_id | raw_customers | Unique identifier for each customer |
| first_name | raw_customers | Customer first name |
| last_name | raw_customers | Customer last name |
| first_order | raw_orders | Date of the customer's very first order |
| most_recent_order | raw_orders | Date of the customer's most recent order |
| number_of_orders | raw_orders | Total count of all orders placed |
| customer_lifetime_value | raw_payments | Sum of all payments made, in dollars |

**Total rows:** 100 (one per customer)
**Location in Snowflake:** `PC_HEVODATA_DB.PUBLIC.CUSTOMERS`


---

## Security

| Practice | Implementation |
|----------|---------------|
| No hardcoded credentials | All secrets passed via environment variables only |
| profiles.yml outside repo | Lives in `~/.dbt/` — never committed to Git |
| .gitignore configured | Blocks credentials, dbt build artifacts, Mac system files |
| Least privilege DB role | The replication DB user has only SELECT and REPLICATION rights |
| Named publication | Hevo can replicate only the three specified tables |
| Credential verification | `git grep` confirms zero literal secrets in the repository |

---


