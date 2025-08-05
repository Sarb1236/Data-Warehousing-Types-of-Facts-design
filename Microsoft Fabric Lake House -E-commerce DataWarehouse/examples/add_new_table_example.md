# Adding New Tables to Configuration

This document shows how to add new tables to the Lake House without modifying the main code.

## Example: Adding a New Table

### Step 1: Add CSV File
Place your new CSV file in `data/raw/` directory:
```
data/raw/
├── customers.csv
├── products.csv
├── categories.csv
├── suppliers.csv
├── locations.csv
├── orders.csv
├── inventory.csv
└── new_table.csv  ← Add your new file here
```

### Step 2: Update Configuration
Add the new table definition to `config/ingestion_config.json`:

```json
{
  "bronze_layer": {
    "tables": {
      "existing_table": { ... },
      "new_table": {
        "source_file": "new_table.csv",
        "description": "New table description",
        "data_quality_rules": {
          "null_checks": ["id", "name", "required_field"],
          "duplicate_checks": ["id", "unique_field"],
          "data_type_validation": {
            "id": "int",
            "amount": "decimal"
          },
          "business_rules": {
            "amount": "> 0",
            "quantity": ">= 0"
          }
        }
      }
    }
  },
  "silver_layer": {
    "transformation_rules": {
      "existing_table": { ... },
      "new_table": {
        "columns": {
          "id": {"cast": "int"},
          "name": {"trim": true, "case": "title"},
          "amount": {"cast": "decimal(10,2)"},
          "created_date": {"date_format": "yyyy-MM-dd"}
        },
        "filters": [
          "id IS NOT NULL",
          "name IS NOT NULL",
          "amount > 0"
        ]
      }
    }
  },
  "gold_layer": {
    "dimensions": {
      "dim_new_table": {
        "source_table": "new_table",
        "columns": ["id", "name", "amount", "created_date"],
        "key_column": "id",
        "key_alias": "new_table_key"
      }
    },
    "facts": {
      "fact_new_table": {
        "source_table": "new_table",
        "joins": [
          {"table": "dim_new_table", "on": "id", "type": "inner"}
        ],
        "columns": ["id", "new_table_key", "amount", "created_date"]
      }
    }
  }
}
```

### Step 3: Run the Notebook
Simply re-run the bronze layer notebook - it will automatically:
- ✅ Detect the new table from configuration
- ✅ Ingest the CSV file to bronze layer
- ✅ Apply data quality rules
- ✅ Generate quality reports

## Configuration Options

### Bronze Layer Options
- `source_file`: CSV filename in data/raw/
- `description`: Table description
- `data_quality_rules`: Quality validation rules

### Silver Layer Options
- `columns`: Column transformations (trim, case, cast, etc.)
- `filters`: Data filtering conditions

### Gold Layer Options
- `dimensions`: Dimension table definitions
- `facts`: Fact table definitions with joins
- `analytical_views`: SQL queries for analytics

## Benefits
- 🚀 **No Code Changes**: Add tables by updating config only
- 🔄 **Automatic Processing**: New tables flow through all layers
- 📊 **Quality Assurance**: Built-in data quality checks
- 📈 **Scalable**: Easy to add hundreds of tables

## Data Quality Rules
- **null_checks**: Columns that must not be null
- **duplicate_checks**: Columns that must be unique
- **data_type_validation**: Expected data types
- **business_rules**: Business logic validation

## Transformation Rules
- **trim**: Remove whitespace
- **case**: Convert case (upper, lower, title)
- **cast**: Convert data types
- **date_format**: Parse date strings
- **regex_replace**: Pattern-based replacements 