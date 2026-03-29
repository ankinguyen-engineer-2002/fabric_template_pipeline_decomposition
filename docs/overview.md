# Metadata-Driven Pipeline Template — Architecture Overview

## Concept

A reusable orchestration template for Microsoft Fabric that follows a single principle: **the pipeline doesn't know what it processes — the metadata table decides.**

Instead of hard-coding table names, notebook paths, or transformation logic into pipelines, everything is driven by a single control table. Adding, removing, or changing a table is a row-level operation — zero pipeline modifications required.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  ┌──────────────┐     ┌──────────┐                                              │
│  │              │     │          │──┐                    ┌── ETL Notebook A      │
│  │  pl_master   │──→  │ pl_brz   │  │   ┌────────────┐  ├── ETL Notebook B      │
│  │  (daily)     │     │ (step 1) │  │   │            │  ├── ETL Notebook C      │
│  │              │     │          │  ├─→ │ env_config  │  ├── ...                 │
│  │  Orchestrator│     ├──────────┤  │   │            │  │                       │
│  │              │──→  │ pl_slv   │  │   │ Shared     │  │  ┌── brz_engine       │
│  │              │     │ (step 2) │──┤   │ Config     │──┤  ├── slv_engine       │
│  │              │     │          │  │   │ Notebook   │  │  └── gld_engine       │
│  │              │     ├──────────┤  │   │            │  │                       │
│  │              │──→  │ pl_gld   │  │   └────────────┘  ├── ETL Notebook X      │
│  │              │     │ (step 3) │──┘                   ├── ETL Notebook Y      │
│  └──────────────┘     └──────────┘                      └── ETL Notebook Z      │
│                                                                                 │
│  ORCHESTRATION         PIPELINES      CONFIG    ENGINES     ETL NOTEBOOKS       │
│  (1 master)            (N layers)     (shared)  (1/layer)   (1/table)           │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Master Pipeline

The entry point. Runs on a daily schedule. Its only job is to trigger child pipelines **sequentially** — each layer waits for the previous one to succeed before starting.

Why sequential: downstream layers depend on upstream data. Gold can't compute KPIs if Silver hasn't finished transforming.

### 2. Layer Pipelines (1 per layer)

Each layer pipeline does the same thing:

```
Lookup (SQL) → read metadata table → get list of tables to process
    ↓
ForEach (parallel) → iterate over the list
    ↓
Call engine notebook with parameters (table_name, notebook_id, load_type)
```

The pipeline itself contains no business logic. It's a generic **ForEach-over-metadata** loop. The same pipeline template works for any layer — what changes is the SQL filter in the Lookup activity (`WHERE layer = 'BRZ'` vs `WHERE layer = 'SLV'`).

**ForEach batchCount** controls parallelism within a layer (typically 3-4 concurrent notebook runs).

### 3. env_config (Shared Configuration Notebook)

A single notebook that every engine calls first. It sets up:

- Lakehouse connection paths
- Workspace/environment detection (Dev vs Prod)
- Shared variables (date ranges, fiscal calendars)
- Logging configuration

**Why it matters**: change a connection string or date range in one place, and all 3 engines pick it up. No need to update individual notebooks.

### 4. Engine Notebooks (1 per layer)

Each layer has one engine notebook. The engine:

1. Receives parameters from the ForEach activity (`table_name`, `notebook_id`, `load_type`)
2. Calls `env_config` to initialize environment
3. Calls the specific ETL notebook identified by `notebook_id`
4. Handles logging, error capture, and status updates back to the metadata table

The engine is the **middleware** — it wraps every ETL run with consistent setup, error handling, and telemetry without duplicating that logic across dozens of ETL notebooks.

### 5. ETL Notebooks (1 per table)

Each table has its own notebook with the actual transformation logic — source queries, joins, filters, aggregations, write-to-Delta. This is where domain-specific business rules live.

ETL notebooks are **pure transforms**: they assume the environment is already configured (by the engine + env_config) and focus solely on "read source → transform → write target."

### 6. Metadata Table

The control plane. A single Delta table with one row per managed table:

| What it stores | Why |
|----------------|-----|
| Table name + layer | Which table, which pipeline processes it |
| Execution order | Dependencies within a layer (e.g., SLV stage 1 before stage 2) |
| Load type | Overwrite vs incremental — the engine uses this to decide the write mode |
| Notebook ID | UUID of the ETL notebook — the engine calls this dynamically |
| is_active flag | Toggle a table on/off without deleting anything |
| Status + row count | Last run outcome — used for monitoring and the visualization |
| Frequency | Daily / Monthly — the Lookup filters by what's due today |

---

## Flow Summary

```
Schedule trigger (daily)
    │
    ▼
pl_master_daily
    │
    ├──→ pl_layer_1  ──→  env_config  ──→  layer_1_engine  ──→  [N ETL notebooks]
    │    (ForEach)         (shared)         (parameterized)       (1 per table)
    │
    ├──→ pl_layer_2  ──→  env_config  ──→  layer_2_engine  ──→  [M ETL notebooks]
    │    (waits for 1)
    │
    └──→ pl_layer_N  ──→  env_config  ──→  layer_N_engine  ──→  [K ETL notebooks]
         (waits for N-1)
```

**Data flows left to right. Control flows top to bottom.**

---

## Medallion Layer Pattern

This template uses the Medallion (multi-hop) pattern, but the architecture itself is layer-agnostic. You could have 2 layers or 10.

| Layer | Role | Typical Load | Transforms |
|-------|------|-------------|------------|
| **REF** | Reference/master data | Overwrite (Monthly) | Minimal — schema alignment only |
| **BRZ** | Raw ingestion | Overwrite or Incremental (Daily) | None — faithful copy of source |
| **SLV** | Business transforms | Overwrite (Daily) | Joins, filters, aggregation, dedup |
| **GLD** | Analytics-ready | Overwrite (Daily) | Final rollups, KPI calculations |

Each layer is processed by its own pipeline + engine pair. The metadata table's `execution_order` column handles dependencies within a layer (e.g., SLV has 3 sequential stages).

---

## Adding or Changing Tables

| Action | What to do | Pipeline change needed? |
|--------|-----------|------------------------|
| Add a table | Create ETL notebook + insert 1 row in metadata table | No |
| Remove a table | Set `is_active = 0` | No |
| Change load type | Update `load_type` column | No |
| Change execution order | Update `execution_order` column | No |
| Add a new layer | Create 1 pipeline + 1 engine notebook + metadata rows | Minimal (copy template) |

---

## Why This Pattern

| Problem | How this template solves it |
|---------|----------------------------|
| Adding a table requires pipeline changes | Metadata-driven: insert a row, done |
| Duplicated setup code across notebooks | `env_config` centralizes all configuration |
| Inconsistent error handling | Engine notebooks wrap every ETL with uniform logging |
| No visibility into what runs and when | Metadata table is the single source of truth; visualization auto-generates from it |
| Hard to parallelize safely | ForEach with batchCount handles concurrency; execution_order handles dependencies |
| Environment-specific configs scattered | `env_config` detects Dev/Prod and sets paths accordingly |
