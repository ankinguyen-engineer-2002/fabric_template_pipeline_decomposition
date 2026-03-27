# 🏛️ Medallion Architecture — ETL Pattern

## Overview

The pipeline follows the **Medallion Architecture** (also called Multi-Hop Architecture) with 4 layers:

```
┌─────────────────────────────────────────────────────────────┐
│                    pl_master_daily                           │
│              Sequential Orchestrator @ 10:00 AM             │
└────────────┬────────────┬────────────┬──────────────────────┘
             │            │            │
             ▼            ▼            ▼
     ┌───────────┐  ┌──────────┐  ┌──────────┐
     │ pl_brz    │→ │ pl_slv   │→ │ pl_gld   │
     │  daily    │  │  daily   │  │  daily   │
     └─────┬─────┘  └─────┬────┘  └─────┬────┘
           │              │              │
           ▼              ▼              ▼
      ┌─────────┐   ┌──────────┐   ┌──────────┐
      │ nb_etl  │   │nb_slv_etl│   │nb_gld_etl│
      │ (Engine)│   │ (Engine) │   │ (Engine) │
      └─────────┘   └──────────┘   └──────────┘
```

## Layers

### 📚 REF — Reference Data (9 tables)
- **Schedule:** Monthly
- **Load Type:** Full Overwrite  
- **Engine:** `nb_etl`
- **Purpose:** Master data — calendars, customers, products, warehouses
- **Example Tables:** `ref_calendar`, `ref_customer_account`, `ref_item_master`, `ref_product`

### 🟠 BRZ — Bronze / Raw Ingestion (7 tables)
- **Schedule:** Daily @ 10:00 AM
- **Load Type:** Overwrite or Incremental  
- **Engine:** `nb_etl`
- **Purpose:** Raw data ingestion from source systems (AFI, CODIS, SupplyChain)
- **Example Tables:** `brz_saleshistory_afi__invoicedetail`, `brz_wholesale_codis_afi__codatan`

### 🟣 SLV — Silver / Transformations (8 tables)
- **Schedule:** Daily  
- **Load Type:** Overwrite
- **Engine:** `nb_slv_etl`
- **Execution Stages:**
  - **Order 2 (Stage 1):** Base transforms — joins, clean, denormalize (3 tables)
  - **Order 3 (Stage 2):** Time-series aggregation — monthly/weekly rollups (4 tables)
  - **Order 4 (Stage 3):** Business models — naive forecasting (1 table)
- **Example Tables:** `slv_invoice_detail_line_level`, `slv_actual_demand_monthly`, `slv_naive_forecast_monthly`

### 🟡 GLD — Gold / Business-Ready (2 tables)
- **Schedule:** Daily
- **Load Type:** Overwrite
- **Engine:** `nb_gld_etl`
- **Purpose:** Final analytics layer — KPIs, forecast vs actual comparisons
- **Tables:** `gld_flat_forecast_actual`, `gld_forecast_kpi_metric`

## Engine Pattern

All layers use a **parameterized engine notebook** pattern:

```
Pipeline (ForEach activity)
  └── For each row in utl_pipeline_metadata:
       └── Call engine notebook with parameters:
            ├── table_name
            ├── notebook_id (specific ETL logic per table)
            └── load_type (overwrite / incremental)
```

This makes the architecture fully **metadata-driven** — adding a new table is as simple as inserting a row into `dbo.utl_pipeline_metadata`.

## Metadata Table: `dbo.utl_pipeline_metadata`

| Column | Type | Description |
|---|---|---|
| `table_name` | string | Target Delta table name |
| `layer` | string | REF / BRZ / SLV / GLD |
| `execution_order` | int | Determines execution sequence within pipeline |
| `load_type` | string | overwrite / incremental |
| `notebook_name` | string | UUID of the ETL notebook for this table |
| `is_active` | bool | Whether this table should be processed |
| `next_run_time` | timestamp | Next scheduled run |
| `row_count` | bigint | Last known row count |
| `status` | string | success / failed |
