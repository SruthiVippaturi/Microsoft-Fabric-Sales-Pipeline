# Microsoft Fabric Data Engineering Pipeline

A production-grade end-to-end data engineering pipeline built on **Microsoft Fabric**, processing daily sales CSV files through a layered architecture (Bronze → Silver → Warehouse) with full audit logging, schema validation, and Power BI reporting.

---

## Architecture Overview

```
Source (CSV)
    ↓
src_product (Source Lakehouse)
    ↓  File validation + Audit log
lhb_Product (Bronze Lakehouse)
    ↓  Schema validation
lhs_product (Silver Lakehouse)
    ↓  Transform & load
Dw_Bi_Reporting (Gold Warehouse)
    ↓
Power BI Dashboard
```

---


---

## Workspace Items

| Item | Type | Purpose |
|------|------|---------|
| `src_product` | Lakehouse | Source CSV landing zone |
| `lhb_Product` | Lakehouse | Bronze — raw data as-is |
| `lhs_product` | Lakehouse | Silver — transformed data |
| `Dw_Bi_Reporting` | Warehouse | Gold — reporting layer |
| `Dw_Bi_Audit` | Warehouse | Audit logging |
| `dev_val_lib` | Variable Library | Environment-specific config |
| `Schema` | Notebook | Schema validation |
| `Transformed` | Notebook | PySpark transformations |
| `Warehouse` | Notebook | SQL warehouse load |
| `PI_Product` | Pipeline | Orchestration |

---

## Pipeline Steps

### Step 1 — Audit Log
Inserts a run record into `Dw_Bi_Audit.dbo.audit_file_run` with status `In Progress`.

### Step 2 — File Check
Validates that exactly one CSV file exists for today's date in the source path:
```
Product/Sales/YYYYMM/sales_ordersYYYYMMDD.csv
```

### Step 3 — Processed File Check
Queries audit table to ensure the file hasn't already been processed (prevents duplicates).

### Step 4 — Copy to Bronze
Copies the raw CSV file as-is to `lhb_Product` lakehouse — no transformations applied.

### Step 5 — Schema Validation
The `Schema` notebook validates columns against the schema definition file stored in:
```
lhb_Product/Files/Schema/sales_order_schema.csv
```

### Step 6 — Transformation
The `Transformed` notebook (PySpark) reads from Bronze, applies business logic and writes a Delta table to `lhs_product`:
- Derived columns (OrderYear, OrderMonth, OrderValueBucket)
- Data type standardization
- Audit timestamp (`Exdate`, `Id`)

### Step 7 — Warehouse Load
The `Warehouse` notebook (T-SQL) loads data from Silver into `Dw_Bi_Reporting.dbo.Sales_Orders_EXCUTED`:
- Deletes existing records for the run date
- Inserts fresh transformed data
- Validates row count after load

### Step 8 — Audit Update
Updates the audit record to `Completed` with end time captured.

---

## Dynamic Configuration — Variable Library

All environment-specific values are managed via `dev_val_lib` (no hardcoded values in pipeline):

| Variable | Description |
|----------|-------------|
| `vl_workspace_id` | Fabric workspace ID |
| `vl_audit_warehouse_id` | Dw_Bi_Audit warehouse ID |
| `vl_source_lakehouse_id` | src_product lakehouse ID |
| `vl_bronze_lakehouse_id` | lhb_Product lakehouse ID |
| `vl_silver_lakehouse_id` | lhs_product lakehouse ID |
| `vl_sqlendpoint` | SQL connection endpoint |
| `vl_Source_Path` | Source folder path |
| `vl_File_name` | File name prefix |
| `vl_nbs_warehouse` | Warehouse notebook ID |
| `vl_nbs_transformed` | Transformed notebook ID |
| `vl_nbb_schema` | Schema notebook ID |
| `vl_env` | Environment (dev/test/prod) |

Supports multi-environment deployment (DEV / TEST / PROD) — same pipeline, no code changes required.

---

## Audit Table Design

Table: `Dw_Bi_Audit.dbo.audit_file_run`

| Column | Description |
|--------|-------------|
| `PipelineRunId` | Unique pipeline run ID |
| `PipelineName` | Name of the pipeline |
| `FileName` | Processed file name |
| `StartTimeUTC` | Pipeline start time |
| `EndTimeUTC` | Pipeline end time |
| `WorkspaceId` | Fabric workspace ID |
| `PipelineStatus` | Completed / Failed / In Progress |
| `Process` | Stage name (Ingestion) |
| `process_date` | File date (YYYYMMDD) |
| `silver_gold_id_status` | Silver/Gold load status |

---

## Error Handling

Every activity has a corresponding Fail activity that:
- Logs the error message and error code
- Calls `dbo.errorlog` stored procedure to write to audit
- Stops pipeline execution gracefully

---

## Target Table Schema

Table: `Dw_Bi_Reporting.dbo.Sales_Orders_EXCUTED`

| Column | Type |
|--------|------|
| OrderID | VARCHAR(50) |
| OrderDate | DATE |
| CustomerID | VARCHAR(50) |
| CustomerName | VARCHAR(100) |
| ProductCategory | VARCHAR(100) |
| ProductName | VARCHAR(150) |
| Quantity | INT |
| UnitPrice | DECIMAL(10,2) |
| Region | VARCHAR(50) |
| SalesChannel | VARCHAR(50) |
| SalesAmount | DECIMAL(12,2) |
| OrderYear | INT |
| OrderMonth | INT |
| OrderValueBucket | VARCHAR(50) |
| Id | BIGINT |
| Exdate | DATETIME2(6) |

---

## Power BI Dashboard

Connected to `Dw_Bi_Reporting` via Direct Lake semantic model (`Sales_Orders_Model`).

Visuals included:
- Total Sales KPI card
- Total Orders KPI card
- Avg Order Value KPI card
- Sales by Region (bar chart)
- Sales by Product Category (bar chart)
- Monthly Sales Trend (line chart)
- Sales by Channel (donut chart)

---

## Technologies Used

- Microsoft Fabric Pipelines
- Microsoft Fabric Lakehouse (OneLake)
- Microsoft Fabric Warehouse (T-SQL)
- PySpark Notebooks
- T-SQL Notebooks
- Variable Library (dynamic config)
- Power BI (Direct Lake)
- Delta Lake format

---

## How to Run

1. Upload daily CSV file to `src_product/Files/Product/Sales/YYYYMM/`
2. File must be named: `sales_ordersYYYYMMDD.csv`
3. Trigger `PI_Product` pipeline manually or via schedule
4. Monitor via Fabric Monitoring Hub
5. Check `Dw_Bi_Audit.dbo.audit_file_run` for run status
6. View results in Power BI dashboard

---

## Key Features

- **Daily ingestion** — processes one file per day
- **Duplicate prevention** — checks audit table before processing
- **Schema validation** — validates columns against schema file
- **Audit logging** — full pipeline run tracking
- **Error handling** — graceful failure with error logging
- **Multi-environment** — DEV/TEST/PROD via variable library
- **No hardcoded values** — fully parameterized pipeline
