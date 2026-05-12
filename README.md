# Hevo Data Engineering Assignment
### Subhankar Kar Chowdhury | Head of Customer Success Candidate

---

## The Objective

This assignment required building a complete, production-grade 
end-to-end ELT pipeline — from a locally hosted PostgreSQL database 
all the way through to a transformed, tested, and documented data 
model in Snowflake. The instructions were intentionally vague, 
requiring independent research, ambiguity resolution, and 
professional documentation alongside the technical build.

The specific deliverables were:

1. A Docker-based PostgreSQL database loaded with three raw datasets
2. A Hevo pipeline using Logical Replication to replicate data 
   into Snowflake in near real time
3. A dbt project that transforms raw data into a clean, 
   tested customers mart table
4. A GitHub repository with no hardcoded credentials, 
   a working README, and a Loom walkthrough

What this document does beyond the checklist: it tells the full 
story of how the pipeline was built, why each decision was made, 
what friction was encountered, and what a real CS team could do 
with this pipeline in production.

---

## Architecture

The complete data flow, from source to mart:
+------------------+ Logical +--------+ COPY +-----------+ dbt +---------------+ | PostgreSQL 16 | Replication | Hevo | -------> | Snowflake | -------> | customers | | (Docker-based | -------------> | ELT | | Warehouse | models | (mart table) | | local machine) | WAL streaming | | | | | | +------------------+ +--------+ +-----------+ +---------------+ raw_customers HEVO_SF_RAW_* CUSTOMERS raw_orders HEVO_SF_RAW_CUSTOMERS - customer_id raw_payments HEVO_SF_RAW_ORDERS - first_name HEVO_SF_RAW_PAYMENTS - last_name - first_order - most_recent_order - number_of_orders - customer_lifetime_value



**Why this architecture?**
Each layer has a single, clear responsibility:
- PostgreSQL owns the raw source data
- Hevo owns the movement of data (no transformation)
- Snowflake owns the storage
- dbt owns all transformation logic

This separation means any layer can be replaced or upgraded 
without breaking the others — a principle that matters deeply 
when managing enterprise customer implementations.

---

## Components — What Each One Does and Why It Was Chosen

### PostgreSQL (Source Database)
PostgreSQL is one of the most widely used open-source relational 
databases in enterprise environments. It was chosen here because:
- It supports Logical Replication natively — essential for 
  real-time CDC (Change Data Capture)
- It is the most common source database type in Hevo customer 
  implementations
- Running it inside Docker means zero installation complexity 
  and complete reproducibility

### Docker
Docker runs PostgreSQL inside a self-contained "box" on your 
local machine. This means:
- No complex database installation required
- The exact same environment runs on any machine
- Easy to start, stop, and reset for testing

### ngrok
Hevo is a cloud service. A local Docker database has no public 
internet address. ngrok creates a secure tunnel — a temporary 
public address pointing to the local database — allowing Hevo 
Cloud to connect to it as if it were hosted on a public server.

This is a real networking challenge that Hevo CS teams encounter 
regularly with customers who have on-premise databases.

### Hevo Data (ELT Pipeline)
Hevo moves data from PostgreSQL to Snowflake automatically, 
in near real time, using Logical Replication. Key reasons for 
Logical Replication over batch ingestion:
- Captures every INSERT, UPDATE, and DELETE as it happens
- Sub-minute latency from source change to Snowflake landing
- No scheduled batch jobs to manage or monitor
- Ideal for customer-facing analytics that need fresh data

### Snowflake (Cloud Data Warehouse)
Snowflake is the destination warehouse where all raw data lands 
and where dbt runs transformations. It was connected via 
Snowflake Partner Connect — which auto-provisions the database, 
warehouse, role, and user for Hevo, removing manual configuration.

### dbt (Data Transformation)
dbt transforms the three raw, scattered tables into one clean, 
business-ready customers table. Unlike writing SQL directly in 
Snowflake, dbt adds:
- Version control — every transformation is a tracked file in Git
- Automated testing — data quality checked on every run
- Auto-generated documentation — every column described and lineaged
- Modular, reusable SQL — no duplicated logic across teams

---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Docker Desktop | Any recent | Runs PostgreSQL locally |
| Python | 3.11 | Required for dbt |
| dbt-snowflake | 1.7+ | dbt adapter for Snowflake |
| ngrok | Any recent | Exposes local DB to Hevo Cloud |
| Snowflake Account | Trial | Cloud data warehouse |
| Hevo Account | Trial via Partner Connect | ELT pipeline |
| Git | Any | Version control |

---

## Setup Instructions

### Step 1: Clone the Repository
```bash
git clone https://github.com/CodeSubho/Hevo-assignment_Subhankar.git
cd Hevo-assignment_Subhankar
```

### Step 2: Set Environment Variables
All credentials are passed via environment variables. 
Never hardcode these in any file.

```bash
# Run these in every new Terminal session before using dbt
export SNOWFLAKE_ACCOUNT="your_snowflake_account"
export SNOWFLAKE_USER="your_snowflake_username"
export SNOWFLAKE_PASSWORD="your_snowflake_password"
```

### Step 3: Set Up the dbt Profile
The profiles.yml lives outside the project to keep 
credentials away from version control:

```bash
mkdir -p ~/.dbt
```

Create ~/.dbt/profiles.yml with this content:
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

### Step 4: Set Up Python Virtual Environment
```bash
cd ~/Desktop
python3.11 -m venv dbt-env
source dbt-env/bin/activate
pip install --upgrade pip
pip install dbt-snowflake
```

### Step 5: Verify dbt Connection
```bash
cd Hevo-assignment_Subhankar
dbt debug
```
All checks should show OK.

### Step 6: Build the Models
```bash
dbt run
```

### Step 7: Run Data Quality Tests
```bash
dbt test
```

---

## The Full Story — How This Was Built, Step by Step

### Chapter 1: Setting Up the Source Database

The starting point was a local PostgreSQL database running 
inside Docker. Docker was chosen over a native installation 
because it is reproducible, isolated, and mirrors how many 
enterprise customers deploy databases in containerised 
environments.

**Why this step matters for CS:**
When onboarding customers to Hevo, the source database 
configuration is almost always where implementation complexity 
lives. Understanding Docker-based deployments, replication 
settings, and publication management is essential for a CS 
leader supporting technical customers.

The database was started with:
```bash
docker run --name hevo-postgres \
  -e POSTGRES_USER=hevo_user \
  -e POSTGRES_PASSWORD=hevo_pass \
  -e POSTGRES_DB=hevo_db \
  -p 5432:5432 \
  -d postgres:15
```

Three tables were created and loaded with the provided 
CSV datasets:
- raw_customers — 100 rows
- raw_orders — 99 rows  
- raw_payments — 113 rows

### Chapter 2: Enabling Logical Replication

This was the most technically critical configuration step — 
and the one most likely to cause failures in real customer 
implementations.

Logical Replication was configured entirely on the PostgreSQL 
side by setting three parameters:

```sql
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;
```

**What these mean:**
- wal_level = logical tells PostgreSQL to keep a detailed 
  log of every data change — not just enough for crash 
  recovery, but enough for external services like Hevo 
  to subscribe to
- max_replication_slots reserves capacity for services 
  that want to subscribe to this log
- max_wal_senders allows concurrent connections to 
  stream the log

A named publication was then created — specifying exactly 
which tables Hevo should replicate:

```sql
CREATE PUBLICATION hevo_publication 
FOR TABLE raw_customers, raw_orders, raw_payments;
```

**Why this step matters for CS:**
This is the step that customers most frequently get wrong. 
A CS leader needs to understand that Hevo cannot configure 
the source database itself — the customer's database team 
must make these changes. Knowing exactly what to ask for, 
and why, is the difference between a smooth onboarding 
and a delayed one.

### Chapter 3: Solving the Networking Challenge

This was the step the assignment left deliberately vague — 
and the one that required the most independent research.

Hevo is a cloud service. A local Docker database has no 
public IP address. The solution was ngrok — a tool that 
creates a secure TCP tunnel from the public internet to 
the local PostgreSQL port:

```bash
ngrok tcp 5432
```

This produced a public forwarding address that was used 
as the Hevo source host — making the local database 
reachable from Hevo Cloud without exposing it permanently 
to the internet.

**Why this step matters for CS:**
Many Hevo customers have on-premise or VPC-hosted databases 
that are not publicly accessible. The networking solution 
varies — ngrok for testing, static IPs with firewall rules 
for production, VPN tunnels for enterprise. A CS leader 
who understands the networking layer can diagnose connection 
failures independently rather than escalating every time.

### Chapter 4: Building the Hevo Pipeline

The Hevo trial was signed up via Snowflake Partner Connect — 
which automatically provisioned the Snowflake destination 
database, warehouse, role, and user. This is the recommended 
path because it eliminates manual Snowflake configuration 
and ensures correct permissions from the start.

The pipeline was configured with:
- Source: PostgreSQL via ngrok tunnel
- Ingestion mode: Logical Replication
- Publication: hevo_publication
- Destination: Snowflake (Partner Connect provisioned)

After the historical load completed, all three tables 
appeared in Snowflake:
- HEVO_SF_RAW_CUSTOMERS — 100 rows
- HEVO_SF_RAW_ORDERS — 99 rows
- HEVO_SF_RAW_PAYMENTS — 113 rows

**Why this step matters for CS:**
The pipeline configuration screen is what most customers 
see during their first Hevo onboarding. A CS leader who 
has personally built a pipeline — including troubleshooting 
the logical replication errors, networking issues, and 
Partner Connect setup — can guide customers through this 
experience with genuine authority.

### Chapter 5: Transforming Data with dbt

With raw data in Snowflake, dbt was used to transform 
three scattered tables into one clean, business-ready 
customers table. The project was structured in three files:

#### models/sources.yml
Defines where the raw source data lives in Snowflake. 
This tells dbt which database and schema to look in, 
and which tables to treat as source data.

#### models/customers.sql
The core transformation logic. This file:
- Joins raw_customers, raw_orders, and raw_payments
- Calculates first_order (earliest order date per customer)
- Calculates most_recent_order (latest order date)
- Counts number_of_orders per customer
- Sums all payments to calculate customer_lifetime_value
- Uses COALESCE to replace NULL with 0 for customers 
  who have not yet placed any orders

#### models/schema.yml
Defines data quality tests that run automatically 
after every build:
- unique test on customer_id — no duplicate customers
- not_null tests on key columns — no missing data
- This is the quality gate that ensures the 
  customers table is always trustworthy

### Chapter 6: Running and Testing

```bash
# Build all models
dbt run
```

dbt run executes the SQL in customers.sql against 
Snowflake and materialises the result as a permanent 
table — not a view, not a temporary result, but a 
real table that any BI tool or analyst can query directly.

```bash
# Validate data quality  
dbt test
```

dbt test runs all tests defined in schema.yml. 
If any test fails — for example, if a duplicate 
customer_id appeared — dbt reports exactly which 
rows failed and why. This makes data quality issues 
immediately visible rather than silently corrupting 
downstream reports.

**Result:** PASS=6, WARN=0, ERROR=0

---

## Project Structure

hevo_subhankar_project/ ├── models/ │ ├── sources.yml ← defines source tables from Hevo/Snowflake │ ├── customers.sql ← transformation logic (the core deliverable) │ └── schema.yml ← data quality tests and column descriptions ├── dbt_project.yml ← project configuration (name, version, paths) ├── .gitignore ← excludes credentials, dbt artifacts, Mac files └── README.md ← this document


---

## Final Output — The Customers Table

The customers mart table produced by dbt contains 
one row per customer with the following structure:

| Column | Source | Description |
|--------|--------|-------------|
| customer_id | raw_customers | Unique identifier for each customer |
| first_name | raw_customers | Customer first name |
| last_name | raw_customers | Customer last name |
| first_order | raw_orders | Date of the customer's very first order |
| most_recent_order | raw_orders | Date of the customer's most recent order |
| number_of_orders | raw_orders | Total count of all orders placed |
| customer_lifetime_value | raw_payments | Sum of all payments in dollars |

**Total rows:** 100 (one per customer)
**Location:** PC_HEVODATA_DB.PUBLIC.CUSTOMERS

---

## What a CS Team Could Do With This Pipeline in Production

This pipeline is not just a technical exercise — it is 
the foundation of a real customer success data system. 
Here is what becomes possible once this is in production:

### 1. Customer Health Scoring
Using customer_lifetime_value, number_of_orders, and 
most_recent_order, a CS team can build an automated 
health score for every customer — flagging accounts 
that haven't ordered recently (churn risk) or whose 
spending has dropped.

### 2. Proactive Churn Prevention
Customers whose most_recent_order was more than 90 days 
ago can be automatically surfaced for CS outreach — 
before they churn, not after.

### 3. Expansion Revenue Identification
Customers with high number_of_orders but low 
customer_lifetime_value may be purchasing lower-value 
products — a signal for upsell conversations.

### 4. Real-Time Dashboards
Because Hevo uses Logical Replication, the Snowflake 
data is never more than minutes old. A Tableau, Looker, 
or Metabase dashboard built on this data reflects 
near-real-time business activity — not yesterday's batch.

### 5. dbt as a Governance Layer
Every transformation is versioned, tested, and documented. 
When a new analyst joins, they can run dbt docs serve and 
immediately understand every table, every column, and 
every data lineage relationship — reducing onboarding 
time and eliminating tribal knowledge.

---

## Security

Security was treated as a first-class concern throughout:

| Practice | Implementation |
|----------|---------------|
| No hardcoded credentials | All secrets passed via environment variables |
| profiles.yml outside repo | Lives in ~/.dbt/ — never committed to Git |
| .env gitignored | Excluded from version control entirely |
| .gitignore configured | Blocks credentials, dbt artifacts, Mac system files |
| Least privilege DB role | hevo_user has only SELECT and REPLICATION rights |
| Named publication | Hevo can only see the three specified tables |

To verify no secrets are present in this repository:
```bash
git grep -r "password" --include="*.yml" --include="*.sql"
```
Only env_var references will appear — never literal values.

---

## Loom Video Walkthrough

A full video walkthrough of this implementation is 
available here: [Add your Loom link here]

The video covers:
- PostgreSQL setup and data loading
- ngrok tunnel configuration
- Hevo pipeline creation and logical replication
- Snowflake data verification
- dbt model build and test results
- GitHub repository structure

---

## Submission Details

- **GitHub:** https://github.com/CodeSubho/Hevo-assignment_Subhankar
- **Hevo Team ID:** [Add your Team ID]
- **Hevo Pipeline ID:** [Add your Pipeline ID]
