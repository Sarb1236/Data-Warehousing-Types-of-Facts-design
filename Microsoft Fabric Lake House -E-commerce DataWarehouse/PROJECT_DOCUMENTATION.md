# Microsoft Fabric Lake House - Multi-Fact E-commerce Project Documentation

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Overview](#architecture-overview)
3. [Data Model](#data-model)
4. [Implementation Details](#implementation-details)
5. [Configuration System](#configuration-system)
6. [Control Tables](#control-tables)
7. [SCD2 Implementation](#scd2-implementation)
8. [Fact Types](#fact-types)
9. [Usage Guide](#usage-guide)
10. [Production Features](#production-features)
11. [Troubleshooting](#troubleshooting)

---

## Project Overview

This project implements a Microsoft Fabric Lake House solution for a multi-fact e-commerce system using a medallion architecture (Bronze to Silver to Gold) with star schema principles. The solution is configuration-driven and production-ready with SCD2 dimensions, multiple fact types, and comprehensive control tables.

### Key Features
- SCD2 (Slowly Changing Dimensions Type 2) for historical tracking
- Multiple Fact Types: Transactional, Accumulating, and Snapshot facts
- Configuration-Driven Architecture for easy extensibility
- Control Tables for audit, watermark, error tracking
- Production-Ready with idempotency, error handling, logging
- Delta Lake for ACID transactions and time travel

---

## Architecture Overview

### Medallion Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   BRONZE LAYER  │    │   SILVER LAYER  │    │    GOLD LAYER   │
│                 │    │                 │    │                 │
│ • Raw CSV Data  │───▶│ • SCD2 Dimensions│───▶│ • Star Schema   │
│ • Data Quality  │    │ • Transformations│    │ • Fact Tables   │
│ • Validation    │    │ • Business Logic │    │ • Analytics     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Data Flow

1. Bronze Layer: Ingests raw CSV files with data quality checks
2. Silver Layer: Applies SCD2 transformations and business logic
3. Gold Layer: Creates star schema with dimensions and facts
4. Control Tables: Track all operations across layers

---

## Data Model

### Input Data (Raw CSV Files)

| Table | Records | Purpose | Key Fields |
|-------|---------|---------|------------|
| `customers.csv` | 15 | Customer master data | customer_id, customer_name, email |
| `products.csv` | 15 | Product catalog | product_id, product_name, unit_price |
| `categories.csv` | 10 | Product categories | category_id, category_name |
| `suppliers.csv` | 10 | Supplier information | supplier_id, supplier_name |
| `locations.csv` | 10 | Store/warehouse locations | location_id, location_name |
| `orders.csv` | 20 | Order transactions | order_id, customer_id, product_id |
| `inventory.csv` | 32 | Inventory snapshots | inventory_id, product_id, location_id |

### Output Data (Gold Layer)

#### Dimension Tables (SCD2)

| Dimension | Purpose | SCD2 Fields | Business Key |
|-----------|---------|-------------|--------------|
| `dim_customer` | Customer master with history | customer_name, email, phone, address, etc. | customer_id |
| `dim_product` | Product catalog with history | product_name, unit_price, category, supplier, etc. | product_id |
| `dim_location` | Location master with history | location_name, address, manager, capacity, etc. | location_id |

#### Fact Tables

| Fact Table | Type | Purpose | Key Metrics |
|------------|------|---------|-------------|
| `fact_order_transaction` | Transactional | Individual order line items | order_amount, quantity, total_amount |
| `fact_order_accumulating` | Accumulating | Customer order summaries | total_orders, total_spent, avg_order_value |
| `fact_inventory_snapshot` | Snapshot | Point-in-time inventory | stock_quantity, available_quantity |

---

## Implementation Details

### Technology Stack

- Platform: Microsoft Fabric Lake House
- Processing: Apache Spark (PySpark)
- Storage: Delta Lake format
- Language: Python
- Configuration: JSON-based
- Control: CSV-based control tables

### Project Structure

```
Multifact Project/
├── config/
│   └── ingestion_config.json     # Central configuration
├── control/
│   ├── audit_log.csv            # Pipeline audit trail
│   ├── watermark.csv            # Last processed tracking
│   ├── error_records.csv        # Error logging
│   └── table_metadata.csv       # Schema tracking
├── data/
│   └── raw/                     # Source CSV files
├── notebooks/
│   ├── 01_bronze_layer.ipynb    # Data ingestion
│   ├── 02_silver_layer.ipynb    # SCD2 transformations
│   └── 03_gold_layer.ipynb      # Star schema creation
└── README.md                    # Project overview
```

---

## Configuration System

### Configuration-Driven Architecture

The entire pipeline is driven by `config/ingestion_config.json`, which defines:

#### Bronze Layer Configuration
```json
{
  "bronze_layer": {
    "tables": {
      "customers": {
        "source_file": "customers.csv",
        "data_quality_rules": {
          "null_checks": ["customer_id", "customer_name", "email"],
          "duplicate_checks": ["customer_id", "email"],
          "data_type_validation": {
            "customer_id": "int",
            "customer_name": "string"
          }
        }
      }
    }
  }
}
```

#### Silver Layer Configuration (SCD2)
```json
{
  "gold_layer": {
    "dimensions": {
      "dim_customer": {
        "scd_type": 2,
        "business_key": "customer_id",
        "scd2_fields": ["customer_name", "email", "phone", "address"],
        "columns": ["customer_id", "customer_name", "email", "phone", "address"]
      }
    }
  }
}
```

#### Gold Layer Configuration
```json
{
  "gold_layer": {
    "facts": {
      "fact_order_transaction": {
        "source_table": "orders",
        "joins": [
          {"table": "dim_customer", "on": "customer_id", "type": "inner"}
        ],
        "columns": ["order_id", "customer_key", "total_amount"]
      }
    }
  }
}
```

### Adding New Tables

To add a new table to the pipeline:

1. Add CSV file to `data/raw/`
2. Update configuration in `config/ingestion_config.json`
3. Re-run notebooks - no code changes needed!

---

## Control Tables

### Audit Log (`control/audit_log.csv`)
Tracks all pipeline operations with timestamps and status.

| Column | Description |
|--------|-------------|
| `run_id` | Unique identifier for each run |
| `table_name` | Table being processed |
| `layer` | Bronze/Silver/Gold |
| `start_time` | Processing start timestamp |
| `end_time` | Processing end timestamp |
| `status` | SUCCESS/FAIL |
| `row_count` | Number of rows processed |
| `error_message` | Error details if failed |

### Watermark (`control/watermark.csv`)
Tracks last processed key/date for incremental loads.

| Column | Description |
|--------|-------------|
| `table_name` | Table name |
| `layer` | Processing layer |
| `last_processed_key` | Last processed business key |
| `last_processed_date` | Last processed date |

### Error Records (`control/error_records.csv`)
Captures failed records with detailed error information.

| Column | Description |
|--------|-------------|
| `run_id` | Associated run identifier |
| `table_name` | Table with error |
| `layer` | Processing layer |
| `error_type` | Type of error (INGESTION/SCD2/etc.) |
| `error_message` | Detailed error message |
| `record_data` | JSON of failed record |
| `timestamp` | Error timestamp |

---

## SCD2 Implementation

### What is SCD2?

Slowly Changing Dimensions Type 2 tracks historical changes in dimension data by:
- Creating new records when attributes change
- Maintaining effective start/end dates
- Using an `is_current` flag to identify current records

### SCD2 Implementation Details

#### 1. Business Key Detection
```python
business_key = dim_cfg['business_key']  # e.g., "customer_id"
```

#### 2. Change Detection
```python
scd2_fields = dim_cfg['scd2_fields']  # Fields to monitor for changes
change_conditions = []
for field in scd2_fields:
    change_conditions.append(f"df_new.{field} != df_existing.{field}")
```

#### 3. Record Management
- New Records: `effective_start_date = current_date()`, `is_current = True`
- Expired Records: `effective_end_date = current_date()`, `is_current = False`
- Unchanged Records: No modification

#### 4. Surrogate Keys
```python
surrogate_key_col = f'{dim_name}_sk'  # e.g., "dim_customer_sk"
df_new = df_new.withColumn(surrogate_key_col, monotonically_increasing_id())
```

### SCD2 Example

**Before Change:**
```
customer_id | customer_name | email           | is_current | effective_start_date | effective_end_date
1          | John Smith    | john@email.com  | True       | 2023-01-15          | 9999-12-31
```

**After Email Change:**
```
customer_id | customer_name | email           | is_current | effective_start_date | effective_end_date
1          | John Smith    | john@email.com  | False      | 2023-01-15          | 2024-01-15
1          | John Smith    | john.new@email.com | True    | 2024-01-15          | 9999-12-31
```

---

## Fact Types

### 1. Transactional Facts (`fact_order_transaction`)

**Purpose**: Individual business events/transactions
**Characteristics**:
- One row per transaction
- Detailed level of granularity
- Used for operational reporting

**Example Metrics**:
- Order amount, quantity, unit price
- Payment method, shipping method
- Order status, timestamps

### 2. Accumulating Facts (`fact_order_accumulating`)

**Purpose**: Aggregated metrics that accumulate over time
**Characteristics**:
- One row per business entity (e.g., customer)
- Aggregated metrics
- Used for analytical reporting

**Example Metrics**:
- Total orders per customer
- Total spent per customer
- Average order value
- Days since first/last order

### 3. Snapshot Facts (`fact_inventory_snapshot`)

**Purpose**: Point-in-time snapshots of business state
**Characteristics**:
- One row per entity at a specific point in time
- Current state information
- Used for trend analysis

**Example Metrics**:
- Stock quantity
- Available quantity
- Reserved quantity
- Reorder levels

---

## Usage Guide

### Prerequisites

1. Microsoft Fabric Workspace with Lake House
2. Apache Spark environment
3. Delta Lake support
4. Python with PySpark

### Step 1: Setup

1. Upload files to your Fabric workspace
2. Create Lake House if not exists
3. Attach notebooks to your Lake House

### Step 2: Run Pipeline

#### Bronze Layer (Data Ingestion)
```python
# Run notebook: 01_bronze_layer.ipynb
# This will:
# - Read CSV files from data/raw/
# - Apply data quality checks
# - Write to bronze layer in Delta format
# - Log to control tables
```

#### Silver Layer (SCD2 Transformations)
```python
# Run notebook: 02_silver_layer.ipynb
# This will:
# - Read from bronze layer
# - Apply SCD2 transformations
# - Create surrogate keys
# - Track historical changes
# - Write to silver layer
```

#### Gold Layer (Star Schema)
```python
# Run notebook: 03_gold_layer.ipynb
# This will:
# - Read from silver layer
# - Create dimension tables (current records only)
# - Create fact tables with joins
# - Build star schema
# - Create analytical views
```

### Step 3: Monitor

Check control tables for:
- Audit Log: Pipeline execution status
- Error Records: Failed operations
- Watermark: Processing progress

### Step 4: Analyze

Query the gold layer tables for:
- Customer Analytics: Lifetime value, segmentation
- Product Analytics: Sales performance, inventory
- Operational Analytics: Order processing, fulfillment

---

## Production Features

### 1. Idempotency
- Safe to re-run notebooks multiple times
- No duplicate data creation
- Consistent results regardless of execution count

### 2. Error Handling
- Graceful failure with detailed error logging
- Failed records captured in error tables
- Pipeline continues processing other tables

### 3. Data Quality
- Null Checks: Ensures required fields are present
- Duplicate Checks: Removes duplicate records
- Data Type Validation: Ensures correct data types
- Business Rules: Validates business logic (e.g., prices > 0)

### 4. Logging & Monitoring
- Audit Trail: Complete operation tracking
- Performance Metrics: Row counts, processing times
- Error Tracking: Detailed error information
- Watermark Tracking: Incremental load support

### 5. Configuration Management
- Centralized Configuration: Single JSON file controls entire pipeline
- Dynamic Table Addition: Add tables without code changes
- Flexible Rules: Configurable data quality and transformation rules

### 6. Scalability
- Delta Lake: ACID transactions and time travel
- Spark Processing: Distributed computing capabilities
- Modular Design: Easy to extend and maintain

---

## Troubleshooting

### Common Issues

#### 1. Configuration Errors
**Problem**: JSON syntax errors in config file
**Solution**: Validate JSON syntax, check for missing commas/brackets

#### 2. File Path Issues
**Problem**: Cannot find CSV files or control tables
**Solution**: Verify file paths in configuration, check file permissions

#### 3. SCD2 Processing Errors
**Problem**: Dimension processing fails
**Solution**: Check business key configuration, verify SCD2 fields exist

#### 4. Memory Issues
**Problem**: Large datasets cause memory errors
**Solution**: Increase Spark memory settings, process in smaller batches

### Debug Steps

1. Check Control Tables: Review audit_log and error_records
2. Validate Configuration: Ensure JSON syntax is correct
3. Test Individual Components: Run notebooks step by step
4. Monitor Spark UI: Check for resource issues
5. Review Logs: Check notebook execution logs

### Performance Optimization

1. Partitioning: Use appropriate partition columns
2. Caching: Cache frequently used DataFrames
3. Broadcast Joins: For small dimension tables
4. Memory Tuning: Adjust Spark memory settings
5. Data Skew: Handle skewed data distributions

---

## Additional Resources

### Microsoft Fabric Documentation
- [Lake House Overview](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview)
- [Delta Lake Guide](https://learn.microsoft.com/en-us/azure/databricks/delta/)
- [Spark SQL Reference](https://spark.apache.org/docs/latest/sql-programming-guide.html)

### Best Practices
- [Data Lake Architecture](https://learn.microsoft.com/en-us/azure/architecture/data-guide/data-lake/)
- [Star Schema Design](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/)
- [SCD Implementation](https://www.kimballgroup.com/2008/05/design-tip-107-slowly-changing-dimension-types-1-2-3/)

### Community Resources
- [Delta Lake Community](https://delta.io/)
- [Apache Spark Community](https://spark.apache.org/community.html)
- [Microsoft Fabric Community](https://community.fabric.microsoft.com/)

---

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review control tables for error details
3. Validate configuration syntax
4. Test with smaller datasets first

---

Your Microsoft Fabric Lake House is now ready for production use with comprehensive documentation! 