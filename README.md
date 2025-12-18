# Medallion Architecture with Spark, dbt, and Databricks on Azure

## Project Overview

This project implements a **Medallion Architecture** (Bronze → Silver → Gold) data lakehouse solution on Azure, transforming raw sales data into analytics-ready datasets using modern data engineering tools.

## Architecture Diagram

```
Azure SQL DB (SalesLT) 
    ↓ [Azure Data Factory]
Bronze Layer (ADLS Gen2 - Parquet files)
    ↓ [Databricks + Spark SQL]
Bronze Tables (Databricks catalog)
    ↓ [dbt + Databricks]
Silver Layer (Delta tables with SCD Type 2)
    ↓ [dbt transformations]
Gold Layer (Analytics marts)
```

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Source** | Azure SQL Database | SalesLT sample database (AdventureWorks) |
| **Ingestion** | Azure Data Factory | Copy data pipeline (SQL → Parquet) |
| **Storage** | Azure Data Lake Storage Gen2 | Bronze layer (`/mnt/bronze`) |
| **Processing** | Azure Databricks + Spark SQL | Table creation from Parquet files |
| **Transformation** | dbt-databricks | Data modeling & orchestration |
| **Orchestration** | VS Code + dbt CLI | Development environment |

## Project Structure

```
medallion-spark-dbt/
├── medallion_spark_dbt/
│   ├── models/
│   │   ├── staging/
│   │   │   └── bronze.yml              # Source definitions
│   │   └── marts/
│   │       └── sales/
│   │           ├── sales.sql           # Gold layer sales mart
│   │           └── sales.yml           # Model documentation
│   ├── snapshots/
│   │   └── address.sql                 # SCD Type 2 for address
│   ├── dbt_project.yml                 # dbt configuration
│   └── profiles.yml                    # Databricks connection
├── databricks_notebooks/               # Spark SQL scripts
└── README.md
```

## Implementation Steps

### 1. Azure SQL Database Setup

````sql
-- Provisioned Azure SQL Database with SalesLT sample database
-- Tables: address, customer, product, salesorderheader, salesorderdetail, etc.
````

### 2. Azure Data Factory Pipeline

**Pipeline Configuration:**
- **Source**: Azure SQL Database (SalesLT schema)
- **Sink**: ADLS Gen2 (`/mnt/bronze/` container)
- **Format**: Parquet files
- **Activity**: Copy Data activity for each table

**Tables Copied:**
- `address`
- `customer`
- `customeraddress`
- `product`
- `productcategory`
- `productdescription`
- `productmodel`
- `salesorderdetail`
- `salesorderheader`

### 3. Databricks Processing (Bronze → Bronze Tables)

Created Databricks notebooks with Spark SQL queries to register Parquet files as Delta tables:

````python
# Example: Creating Bronze tables from Parquet
spark.sql("""
    CREATE TABLE IF NOT EXISTS saleslt.address
    USING PARQUET
    LOCATION '/mnt/bronze/address/'
""")

spark.sql("""
    CREATE TABLE IF NOT EXISTS saleslt.product
    USING PARQUET
    LOCATION '/mnt/bronze/product/'
""")

# Repeat for all 9 tables
````

**Result**: Bronze tables available in Databricks catalog under `saleslt` schema.

### 4. Local Development Setup (VS Code)

#### Installation

````powershell
# Install Python dependencies
pip install dbt-databricks databricks-cli

# Configure Databricks CLI
databricks configure --token
# Enter: Databricks workspace URL and personal access token
````

#### dbt Project Initialization

````powershell
# Initialize dbt project
dbt init medallion_spark_dbt

# Test connection
dbt debug
````

### 5. dbt Configuration

#### [`dbt_project.yml`](dbt_project.yml )

````yaml
name: 'medallion_spark_dbt'
version: '1.0.0'
config-version: 2

profile: 'medallion_spark_dbt'

model-paths: ["models"]
snapshot-paths: ["snapshots"]
target-path: "target"

models:
  medallion_spark_dbt:
    staging:
      +materialized: view
    marts:
      +materialized: table
      +file_format: delta
````

#### [`profiles.yml`](profiles.yml )

````yaml
medallion_spark_dbt:
  target: dev
  outputs:
    dev:
      type: databricks
      host: <your-databricks-workspace-url>
      http_path: /sql/1.0/warehouses/<warehouse-id>
      token: <your-personal-access-token>
      schema: saleslt
      threads: 4
````

### 6. Source Configuration - Bronze Layer

#### [`models/staging/bronze.yml`]bronze.yml )

````yaml
version: 2

sources:
  - name: saleslt
    schema: saleslt
    description: "AdventureWorks database loaded into bronze layer"
    tables:
      - name: address
      - name: customer
      - name: customeraddress
      - name: product
      - name: productcategory
      - name: productdescription
      - name: productmodel
      - name: salesorderdetail
      - name: salesorderheader
````

**Purpose**: References Bronze tables in Databricks for dbt transformations.

### 7. Silver Layer - SCD Type 2 Snapshots

#### [`snapshots/address.sql`](snapshots/address.sql )

````sql
{% snapshot address_snapshot %}
{{
    config(
      file_format = "delta",
      location_root = "/mnt/silver/address",
      target_schema='snapshots',
      invalidate_hard_deletes=True,
      unique_key='AddressID',
      strategy='check',
      check_cols='all'
    )
}}

select
    AddressID,
    AddressLine1,
    AddressLine2,
    City,
    StateProvince,
    CountryRegion,
    PostalCode
from {{ source('saleslt', 'address') }}

{% endsnapshot %}
````

**Features:**
- **Delta format** for ACID transactions
- **SCD Type 2** change tracking (preserves history)
- **Automatic metadata**: `dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id`

### 8. Gold Layer - Sales Mart

#### [`models/marts/sales/sales.sql`](models/marts/sales/sales.sql )

````sql
{{
    config(
        materialized = "table",
        file_format = "delta",
        location_root = "/mnt/gold/sales"
    )
}}

with salesorderdetail_snapshot as (
    select * from {{ ref("salesorderdetail_snapshot") }}
),

product_snapshot as (
    select * from {{ source('saleslt', 'product') }}
),

salesorderheader_snapshot as (
    select 
        *,
        row_number() over (partition by SalesOrderID order by SalesOrderID) as row_num
    from {{ source('saleslt', 'salesorderheader') }}
)

select
    sod.SalesOrderID,
    sod.SalesOrderDetailID,
    sod.OrderQty,
    sod.ProductID,
    sod.UnitPrice,
    sod.LineTotal,
    p.Name as ProductName,
    p.ProductNumber,
    p.StandardCost,
    p.ListPrice,
    soh.OrderDate,
    soh.CustomerID,
    soh.SubTotal,
    soh.TaxAmt,
    soh.TotalDue
from salesorderdetail_snapshot sod
left join product_snapshot p on sod.ProductID = p.ProductID
left join salesorderheader_snapshot soh on sod.SalesOrderID = soh.SalesOrderID
where soh.row_num = 1
````

**Purpose**: 
- Denormalized fact table for sales analytics
- Combines transaction details with product and order dimensions
- Removes duplicate headers using window functions

#### [`models/marts/sales/sales.yml`](models/marts/sales/sales.yml )

````yaml
version: 2

models:
  - name: sales
    description: "Fact table for sales transactions"
    columns:
      - name: SalesOrderID
        tests:
          - unique
          - not_null
      - name: ProductID
        tests:
          - not_null
      - name: OrderDate
        tests:
          - not_null
      # ... additional column definitions with tests
````

**Features**: Data quality tests (uniqueness, nullability checks)

## Medallion Layers Explained

### 🥉 Bronze Layer
- **Format**: Parquet files → Delta tables
- **Location**: `/mnt/bronze/` (ADLS Gen2)
- **Purpose**: Raw, unprocessed data from source systems
- **Schema**: `saleslt` schema in Databricks

### 🥈 Silver Layer
- **Format**: Delta tables with SCD Type 2
- **Location**: `/mnt/silver/` (ADLS Gen2)
- **Purpose**: Cleaned, deduplicated, with change tracking
- **Features**: Historical snapshots, slowly changing dimensions

### 🥇 Gold Layer
- **Format**: Delta tables (marts)
- **Location**: `/mnt/gold/` (ADLS Gen2)
- **Purpose**: Analytics-ready, business-level aggregations
- **Features**: Denormalized for BI tools, optimized queries

## Running the Project

### Execute dbt Commands

````powershell
# Install dependencies
dbt deps

# Test connection
dbt debug

# Run all models
dbt run

# Run snapshots (Silver layer)
dbt snapshot

# Test data quality
dbt test

# Generate documentation
dbt docs generate
dbt docs serve
````

### Development Workflow

````powershell
# 1. Develop models in VS Code
# 2. Compile to check SQL syntax
dbt compile --select sales

# 3. Run specific model
dbt run --select sales

# 4. Test model
dbt test --select sales

# 5. Run full pipeline (snapshots + models)
dbt snapshot && dbt run && dbt test
````

## Data Lineage

```
Azure SQL (saleslt.address)
  → ADF Pipeline
    → ADLS Gen2 (/mnt/bronze/address.parquet)
      → Databricks (saleslt.address table)
        → dbt snapshot (snapshots.address_snapshot)
          → Gold mart (sales.sql)
```

**Tracked by dbt**: Run `dbt docs serve` to visualize complete lineage graph.

## Key Features

### ✅ Implemented
- [x] End-to-end Azure data pipeline
- [x] Medallion architecture (Bronze/Silver/Gold)
- [x] Delta Lake format for ACID transactions
- [x] SCD Type 2 historical tracking
- [x] Data quality tests in dbt
- [x] Modular SQL transformations
- [x] Version-controlled data models

### 🔄 Extensibility
- Add more snapshots for `customer`, `product`, etc.
- Implement incremental models for large tables
- Add dbt macros for reusable logic
- Schedule dbt runs with Databricks Jobs or Airflow
- Add data freshness checks

## Dependencies

````yaml
# Python packages
dbt-databricks==1.7.0
databricks-cli==0.18.0

# Azure Resources
- Azure SQL Database
- Azure Data Factory
- Azure Data Lake Storage Gen2
- Azure Databricks (with SQL Warehouse)
````

## Best Practices Followed

1. **Separation of Concerns**: Bronze (raw) → Silver (cleaned) → Gold (aggregated)
2. **Version Control**: All SQL transformations in Git
3. **Testing**: dbt tests for data quality validation
4. **Documentation**: YAML files document all models
5. **Incremental Processing**: Snapshots prevent full table scans
6. **Delta Format**: ACID compliance, time travel, schema evolution

## Troubleshooting

### Common Issues

**Connection Error:**
````powershell
# Test Databricks connection
dbt debug

# Verify credentials in profiles.yml
````

**Source Not Found:**
````powershell
# Ensure Bronze tables exist in Databricks
spark.sql("SHOW TABLES IN saleslt")
````

**Snapshot Fails:**
````powershell
# Check if snapshot schema exists
spark.sql("CREATE SCHEMA IF NOT EXISTS snapshots")
````

## Next Steps

1. **Add More Snapshots**: Track changes in `product`, `customer`, `salesorderheader`
2. **Create Additional Marts**: Customer analytics, product performance
3. **Implement Incremental Models**: For large, append-only tables
4. **Set Up CI/CD**: GitHub Actions for automated testing
5. **Add Orchestration**: Schedule dbt runs with Databricks Workflows

---

**Documentation Generated by dbt**: Run `dbt docs serve` for interactive project documentation with lineage graphs.
