# Pipeline Template Overview — Enterprise SupplyChain

## 1. What This System Does

A **metadata-driven ETL orchestration system** on Microsoft Fabric that:

- Ingests raw data from 5+ source systems (AFI Sales, CODIS Wholesale, SupplyChain DW)
- Transforms through 4 layers (REF → BRZ → SLV → GLD) into business-ready analytics
- Orchestrates 26 tables across 3 sequential pipelines, triggered daily at 10:00 AM
- Self-documents via an auto-generated interactive D3.js decomposition tree

---

## 2. Execution Flow

```
                                    ┌── brz_engine ── REF 9 ETL notebooks ── [9 ref tables]
pl_master_daily ── ① pl_brz_daily ──┤
     │                               └── brz_engine ── BRZ 7 ETL notebooks ── [7 brz tables]
     │
     │             ② pl_slv_daily ──┐
     │                               ├── slv_engine ── Stage 1 (Order 2) ── [3 slv tables]
     ├─────────────────────────────── ├── slv_engine ── Stage 2 (Order 3) ── [4 slv tables]
     │                               └── slv_engine ── Stage 3 (Order 4) ── [1 slv table]
     │
     └──────────── ③ pl_gld_daily ──── gld_engine ── [2 gld tables]
```

**Key point**: All 3 engines call `env_config` notebook first for environment setup (connection strings, Lakehouse paths, shared variables). This is the single source of truth for runtime configuration.

### Execution Order

| Step | Pipeline | Engine | Tables | Depends On |
|------|----------|--------|--------|------------|
| 1 | `pl_brz_daily` | `brz_engine` | 9 REF + 7 BRZ = 16 | None (first) |
| 2 | `pl_slv_daily` | `slv_engine` | 8 SLV (3 stages) | Step 1 success |
| 3 | `pl_gld_daily` | `gld_engine` | 2 GLD | Step 2 success |

Sequential — if Step 1 fails, Steps 2 and 3 do not run.

---

## 3. Architecture Components

### 3.1 Pipelines

| Pipeline | Schedule | Pattern | batchCount |
|----------|----------|---------|------------|
| `pl_master_daily` | Daily 10:00 AM SEA | Sequential orchestrator — triggers 3 child pipelines | — |
| `pl_brz_daily` | Triggered by master | Lookup → ForEach → Engine call | 3 |
| `pl_slv_daily` | Triggered by master | 3x (Lookup → ForEach) per execution order | 4 |
| `pl_gld_daily` | Triggered by master | Lookup → ForEach → Engine call | 3 |

**ForEach pattern**: each pipeline reads `dbo.utl_pipeline_metadata` via a Lookup activity, then iterates over matching rows and calls the engine notebook for each table.

### 3.2 Notebooks

There are 3 types of notebooks in this system:

#### Type 1: `env_config` (1 notebook, shared)
- Called by all 3 engine notebooks at startup
- Sets up: Lakehouse connection, workspace paths, date variables, logging config
- Single source of truth for runtime environment
- Notebook ID: `4e51e622-77f4-4b80-8028-5e4d1b38a265`

#### Type 2: Engine Notebooks (3 notebooks, one per layer)

| Engine | Layer(s) | Purpose |
|--------|----------|---------|
| `brz_engine` | REF + BRZ | Calls `env_config`, then runs the ETL notebook passed as parameter |
| `slv_engine` | SLV | Calls `env_config`, then runs the ETL notebook with join/transform logic |
| `gld_engine` | GLD | Calls `env_config`, then runs the ETL notebook for final aggregation |

Engines are **parameterized** — they receive `table_name`, `notebook_id`, `load_type` from the ForEach activity and route execution to the correct ETL notebook.

#### Type 3: ETL Notebooks (26 notebooks, one per table)

Each table has its own dedicated notebook containing the specific transformation logic:

- **REF ETL notebooks** (9): full overwrite from source, minimal transforms (e.g., `nb_ref_calendar_2`, `nb_ref_product_2`)
- **BRZ ETL notebooks** (7): raw ingestion from AFI/CODIS/DW sources (e.g., `nb_brz_SalesHistory_AFI__InvoiceDetail_2`)
- **SLV ETL notebooks** (8): joins, filters, aggregations across BRZ/REF tables (e.g., `nb_slv_forecast_demand_monthly_2`)
- **GLD ETL notebooks** (2): final business-ready rollups (e.g., `nb_gld_flat_forecast_actual_2`)

### 3.3 Metadata Table: `dbo.utl_pipeline_metadata`

The **control table** that drives all orchestration. Located in `SupplyChain_Lakehouse`.

| Column | Type | Description |
|--------|------|-------------|
| `table_name` | string | Target Delta table (e.g., `ref_calendar_2`) |
| `layer` | string | `REF` / `BRZ` / `SLV` / `GLD` |
| `execution_order` | int | Sequence: 1 (REF+BRZ) → 2,3,4 (SLV stages) → 5 (GLD) |
| `load_type` | string | `overwrite` or `incremental` |
| `frequency` | string | `Daily` or `Monthly` |
| `notebook_name` | string | UUID of the ETL notebook for this table |
| `is_active` | int | 1 = process, 0 = skip |
| `rows_loaded` | bigint | Row count from last successful run |
| `status` | string | `success` / `failed` |
| `error_message` | string | Error details if failed |
| `pipeline_notes` | string | Lineage trace (source → joins → output) |
| `last_load_date` | timestamp | When last run completed |
| `next_run_time` | timestamp | Next scheduled execution |

**Current state**: 26 active rows, ~233M total rows across all tables.

To add a new table: insert 1 row into this table with the correct layer, order, and notebook UUID. The pipeline picks it up automatically on next run.

---

## 4. Medallion Layers

### REF — Reference Data (9 tables, Order 1, Monthly)

Static/slow-changing master data. Full overwrite each run.

| Table | Rows | Description |
|-------|------|-------------|
| `ref_calendar_2` | 21K | Fiscal + calendar date dimensions |
| `ref_customer_account_2` | 36K | Customer account master |
| `ref_customer_grouping_2` | 35K | Customer segmentation |
| `ref_customer_shipping_location_2` | 127K | Ship-to addresses |
| `ref_forecast_horizon_2` | 5 | Forecast period definitions |
| `ref_item_master_2` | 379K | Product/SKU master |
| `ref_order_type_2` | 29 | Order type codes |
| `ref_product_2` | 373K | Extended product attributes |
| `ref_warehouse_2` | 55 | Warehouse locations |

### BRZ — Bronze Raw Ingestion (7 tables, Order 1, Daily)

Raw data from source systems, minimal transformation.

| Table | Rows | Source | Load |
|-------|------|--------|------|
| `brz_saleshistory_afi__invoicedetail_2` | 36.3M | AFI Sales | Overwrite |
| `brz_saleshistory_afi__invoiceheader_2` | 4.1M | AFI Sales | Overwrite |
| `brz_supplychain_enh_1__demandforecastsnapshotdaily_2` | 0 | SupplyChain DW | Incremental |
| `brz_wholesale_codis_afi__codatan_2` | 918K | CODIS Wholesale | Overwrite |
| `brz_wholesale_codis_afi__comast_2` | 232K | CODIS Wholesale | Overwrite |
| `brz_wholesale_codis_afi__extord_2` | 232K | CODIS Wholesale | Overwrite |
| `brz_wholesale_codis_afi__extorit_2` | 912K | CODIS Wholesale | Overwrite |

### SLV — Silver Transforms (8 tables, Orders 2-3-4, Daily)

Business logic: joins, filters, time-series aggregation.

| Stage | Order | Table | Rows | Logic |
|-------|-------|-------|------|-------|
| 1 | 2 | `slv_forecast_demand_monthly_2` | 13.9M | BRZ forecast → REF cycle join → calendar filter |
| 1 | 2 | `slv_invoice_detail_line_level_2` | 85.8M | BRZ invoice detail+header → REF customer grouping |
| 1 | 2 | `slv_open_order_line_level_2` | 290K | BRZ CODIS tables → REF item+order type joins |
| 2 | 3 | `slv_actual_demand_monthly_2` | 6.0M | SLV invoice → REF calendar → fiscal grouping |
| 2 | 3 | `slv_actual_demand_weekly_2` | 14.4M | SLV invoice → REF calendar → weekly rollup |
| 2 | 3 | `slv_invoice_weekly_2` | 36.3M | SLV invoice → REF calendar → weekly agg |
| 2 | 3 | `slv_open_order_monthly_2` | 122K | SLV open order → REF calendar → monthly rollup |
| 3 | 4 | `slv_naive_forecast_monthly_2` | 5.0M | SLV actuals → calendar weeks → naive model |

**Dependency**: Stage 2 depends on Stage 1 output. Stage 3 depends on Stage 2 output.

### GLD — Gold Business-Ready (2 tables, Order 5, Daily)

Final analytics tables consumed by Power BI reports.

| Table | Rows | Purpose |
|-------|------|---------|
| `gld_flat_forecast_actual_2` | 24.9M | Forecast vs actual comparison — flat denormalized |
| `gld_forecast_kpi_metric_2` | 2.1M | KPI metrics: accuracy, bias, MAPE by segment |

---

## 5. Call Chain (Runtime)

When `pl_master_daily` triggers at 10:00 AM, the exact call sequence is:

```
1. pl_master_daily starts
2.   ├── pl_brz_daily starts
3.   │     ├── Lookup: SELECT * FROM utl_pipeline_metadata WHERE layer IN ('REF','BRZ') AND is_active=1
4.   │     └── ForEach (batchCount=3):
5.   │           └── brz_engine(table_name, notebook_id, load_type)
6.   │                 ├── %run env_config        ← shared config
7.   │                 └── %run {notebook_id}     ← table-specific ETL
8.   │
9.   ├── pl_slv_daily starts (after pl_brz succeeds)
10.  │     ├── Lookup Order 2 → ForEach → slv_engine → env_config → ETL notebook
11.  │     ├── Lookup Order 3 → ForEach → slv_engine → env_config → ETL notebook
12.  │     └── Lookup Order 4 → ForEach → slv_engine → env_config → ETL notebook
13.  │
14.  └── pl_gld_daily starts (after pl_slv succeeds)
15.        ├── Lookup Order 5 → ForEach → gld_engine → env_config → ETL notebook
16.        └── Done
```

**Total runtime**: ~25-30 minutes for all 26 tables (~233M rows).

---

## 6. Visualization

An interactive D3.js horizontal tree diagram is auto-generated from the metadata table.

**Live demo**: https://ankinguyen-engineer-2002.github.io/fabric_template_pipeline_decomposition/

### Visual Layout

```
pl_master ──→ ① pl_brz ──┐                    ┌──→ brz_engine ──→ [REF ETLs] [BRZ ETLs]
              ② pl_slv ──┤──→ env_config ──→ ──┤──→ slv_engine ──→ [SLV Stage 1/2/3]
              ③ pl_gld ──┘     (hub)           └──→ gld_engine ──→ [GLD ETLs]
```

### Node Types

| Type | Visual | Color | Example |
|------|--------|-------|---------|
| Master pipeline | Blue glow border, centered text | `#3b82f6` | `pl_master_daily` |
| Child pipeline | Layer-colored fill + accent bar | BRZ=`#f97316`, SLV=`#a855f7`, GLD=`#eab308` | `① pl_brz_daily` |
| Config notebook | Teal dashed border, glow | `#14b8a6` | `env_config` |
| Engine notebook | Layer-colored dashed border | Same as pipeline | `brz_engine` |
| Table group | Subtle border + accent bar | Same as layer | `REF · 9 ETL notebooks` |
| ETL notebook | Dark fill, status dot | Same as layer | `calendar` (22K rows) |

### Interaction
- **Scroll** to zoom, **drag** to pan
- **Hover** any node for detailed tooltip (rows, status, runtime, lineage)
- **Sidebar** shows collapsible layer panels with all tables

### How to Regenerate

Say **"scan fabric"** in Claude Code. The automated workflow will:
1. Connect to Azure via `az rest`
2. Download latest `utl_pipeline_metadata` from OneLake Delta table
3. Update the `const DATA=[...]` block in `index.html`
4. Preview on localhost
5. Push to GitHub Pages

Only the DATA array and sidebar stats are updated. The layout, hierarchy, bus pattern, and styling are NOT modified during a refresh.

---

## 7. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Metadata-driven orchestration | Adding a table = 1 row insert, no pipeline changes |
| Engine notebook pattern | Shared logic (logging, error handling) in 3 engines, specific transforms in 26 ETL notebooks |
| `env_config` as shared config | Single source of truth — change once, applies everywhere |
| Sequential pipeline execution | Data dependencies: GLD needs SLV, SLV needs BRZ |
| ForEach with batchCount | Parallel within layer (3-4 concurrent), sequential between layers |
| Delta Lake (Parquet + transaction log) | ACID transactions, time travel, efficient incremental reads |
| D3.js visualization | Interactive, zoomable, no external dependencies, works as static HTML |

---

## 8. Adding a New Table

1. Create ETL notebook with the transform logic (e.g., `nb_slv_new_table_2`)
2. Note the notebook UUID from Fabric workspace
3. Insert row into `dbo.utl_pipeline_metadata`:

```sql
INSERT INTO dbo.utl_pipeline_metadata
(table_name, layer, execution_order, load_type, frequency,
 notebook_name, is_active, scheduled_hour)
VALUES
('slv_new_table_2', 'SLV', 3, 'overwrite', 'Daily',
 '<notebook-uuid>', 1, 2)
```

4. Run **"scan fabric"** to update the visualization
5. The pipeline automatically picks up the new table on next scheduled run

No pipeline modifications, no deployment, no code changes to engines.
