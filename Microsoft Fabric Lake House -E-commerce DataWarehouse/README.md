# Microsoft Fabric Lake House - Multi-Fact E-commerce Project

## Project Overview
This project implements a comprehensive Microsoft Fabric Lake House solution for an e-commerce business using star schema design patterns. The solution includes multiple fact tables (transactional, accumulating, and snapshot facts) with dimension tables to support comprehensive analytics and reporting.

## Architecture Components

### 1. Data Sources
- Transactional Data: Orders, order items, payments, shipping
- Customer Data: Customer profiles, demographics, preferences
- Product Data: Product catalog, categories, inventory
- Time Data: Date/time dimensions for temporal analysis

### 2. Dimension Tables
- DimCustomer: Customer information and demographics
- DimProduct: Product catalog and attributes
- DimDate: Time dimension for temporal analysis
- DimLocation: Geographic and shipping information
- DimCategory: Product categorization hierarchy

### 3. Fact Tables
- FactOrderTransaction: Transactional facts for order events
- FactOrderAccumulating: Accumulating facts for customer lifetime value
- FactInventorySnapshot: Snapshot facts for inventory levels
- FactCustomerSnapshot: Snapshot facts for customer status

## Project Structure
```
├── README.md
├── notebooks/
│   ├── 01_data_ingestion.ipynb
│   ├── 02_dimension_tables.ipynb
│   ├── 03_fact_tables.ipynb
│   ├── 04_data_quality.ipynb
│   └── 05_analytics_dashboard.ipynb
├── sql/
│   ├── create_tables.sql
│   ├── sample_data.sql
│   └── analytics_queries.sql
├── config/
│   └── lakehouse_config.json
└── data/
    ├── raw/
    ├── bronze/
    ├── silver/
    └── gold/
```

## Implementation Steps
1. Data Ingestion: Load raw data into bronze layer
2. Data Processing: Transform and clean data in silver layer
3. Star Schema Creation: Build dimension and fact tables in gold layer
4. Data Quality: Implement data quality checks
5. Analytics: Create analytical queries and dashboards

## Technologies Used
- Microsoft Fabric Lake House
- Apache Spark (PySpark)
- Delta Lake
- SQL Serverless
- Power BI (for visualization)

## Getting Started
1. Create a Microsoft Fabric workspace
2. Set up a Lake House
3. Upload the notebooks to your workspace
4. Execute notebooks in sequence
5. Connect Power BI for visualization 