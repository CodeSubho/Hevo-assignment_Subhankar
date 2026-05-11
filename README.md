# Hevo Data Engineering Assignment
## Subhankar Kar Chowdhury

## Overview
End-to-end ELT pipeline built for the Hevo Customer Success take-home assignment:
- Source: PostgreSQL (Docker) with raw_customers, raw_orders, raw_payments
- Pipeline: Hevo Data (Logical Replication mode)
- Destination: Snowflake
- Transformation: dbt (customers mart model)

## Architecture
## PostgreSQL (Docker) → Hevo Pipeline → Snowflake → dbt → customers table


## Prerequisites
- Docker Desktop installed
- Python 3.11 installed
- dbt-snowflake installed
- Snowflake trial account
- Hevo trial account

## Setup Instructions

### Step 1: Set Environment Variables
Before running anything, set these in your terminal:
```bash
export SNOWFLAKE_ACCOUNT="your_snowflake_account"
export SNOWFLAKE_USER="your_snowflake_username"
export SNOWFLAKE_PASSWORD="your_snowflake_password"
```

### Step 2: Set Up dbt Profile
Copy the example profile and place it outside the project:
```bash
mkdir -p ~/.dbt
cp profiles.yml.example ~/.dbt/profiles.yml
```

### Step 3: Activate Virtual Environment
```bash
source ~/Desktop/dbt-env/bin/activate
```

### Step 4: Install Dependencies
```bash
pip install dbt-snowflake
dbt deps
```

### Step 5: Test Connection
```bash
dbt debug
```

### Step 6: Build Models
```bash
dbt run
```

### Step 7: Run Tests
```bash
dbt test
```

## Project Structure
## models sources.yml ← defines source tables from Hevo
## models schema.yml ← dbt tests for data quality
## models customers.sql ← final customers mart model

## Final customers Table Columns
| Column | Description |
|--------|-------------|
| customer_id | Unique customer identifier |
| first_name | Customer first name |
| last_name | Customer last name |
| first_order | Date of first order |
| most_recent_order | Date of most recent order |
| number_of_orders | Total number of orders |
| customer_lifetime_value | Total amount spent |

## Security
- No credentials are hardcoded anywhere in this project
- All sensitive values are passed via environment variables
- .env file is gitignored and never committed
