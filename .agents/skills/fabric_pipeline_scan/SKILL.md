---
name: fabric_pipeline_scan
description: Scan a Microsoft Fabric workspace via MCP Fabric tools, extract pipeline metadata from utl_pipeline_metadata Delta table, decode pipeline definitions, and generate an interactive D3.js decomposition tree visualization. Run this end-to-end automatically.
---

# Fabric Pipeline Decomposition — Automated Scan & Visualize

This skill scans a Fabric workspace, extracts all pipeline/notebook/table metadata, and generates a professional D3.js horizontal decomposition tree.

## When to Use

- User asks to "scan fabric pipelines", "vẽ luồng pipeline", "generate pipeline diagram", or similar
- User mentions `pl_master_daily`, `utl_pipeline_metadata`, or pipeline decomposition

## Configuration

The default workspace is:

```
WORKSPACE_NAME = "Enterprise SupplyChain-Dev"
WORKSPACE_ID   = "c8d9fc83-18b6-4e1d-8264-0b49eed36fe0"
LAKEHOUSE_NAME = "SupplyChain_Lakehouse"
LAKEHOUSE_ID   = "62a3081e-4093-4f46-856c-f50aa58732fa"
METADATA_TABLE = "dbo.utl_pipeline_metadata"
OUTPUT_FILE    = "pipeline_workflow.html"
```

If the user specifies a different workspace, adapt accordingly. If no workspace is specified, use the defaults above.

---

## EXECUTION STEPS

Execute these steps sequentially. Each step specifies the exact MCP tool/command to call. Do NOT skip steps — each one builds on the previous. Log progress after each phase.

---

### PHASE 1: WORKSPACE DISCOVERY

#### Step 1 — List workspaces (if workspace ID not yet known)

```
Tool:    mcp_fabric_onelake
Command: onelake_list_workspaces
Params:  {}
```

Find the target workspace name → save workspace ID.

#### Step 2 — List all items in workspace

```
Tool:    mcp_fabric_onelake
Command: onelake_list_items
Params:  { "workspace": "<WORKSPACE_ID>" }
```

From the results, identify and save:

| What to Find | Item Type | Save As |
|---|---|---|
| Master pipeline | DataPipeline named `pl_master*` | `MASTER_PIPELINE_ID` |
| Bronze pipeline | DataPipeline named `pl_brz*` | `BRZ_PIPELINE_ID` |
| Silver pipeline | DataPipeline named `pl_slv*` | `SLV_PIPELINE_ID` |
| Gold pipeline | DataPipeline named `pl_gld*` | `GLD_PIPELINE_ID` |
| Lakehouse | Lakehouse item | `LAKEHOUSE_ID` |
| Engine notebooks | Notebooks named `nb_etl`, `nb_slv_etl`, `nb_gld_etl` | `ENGINE_NOTEBOOKS` |

#### Step 3 — List Lakehouse tables

```
Tool:    mcp_fabric_onelake
Command: onelake_list_files
Params:  {
  "workspace": "<WORKSPACE_ID>",
  "item": "<LAKEHOUSE_NAME>.Lakehouse",
  "path": "Tables"
}
```

Confirm `dbo.utl_pipeline_metadata` exists in the Tables folder.

---

### PHASE 2: PIPELINE DEFINITION EXTRACTION

#### Step 4 — Get Pipeline API spec (for reference)

```
Tool:    mcp_fabric_docs
Command: docs_get-openapi-spec
Params:  { "workload-type": "dataPipeline" }
```

This tells you how pipeline definitions are structured. The key endpoint is `getDefinition` which returns base64-encoded JSON.

#### Step 5 — Download each sub-pipeline definition

For each pipeline (BRZ, SLV, GLD), the definition was previously retrieved. The definitions contain base64-encoded `pipeline-content.json` payloads. Decode them to extract:

- **Activity structure**: Lookup → ForEach → Engine notebook call
- **SQL queries**: What each Lookup reads from `utl_pipeline_metadata`
- **ForEach config**: `batchCount`, inner activity name
- **Engine notebook ID**: Which notebook is called per table

**Expected pipeline patterns:**

| Pipeline | Lookup Query Filter | Engine | batchCount |
|---|---|---|---|
| `pl_brz_daily` | `layer IN ('BRZ','REF') AND execution_order = 1` | `nb_etl` | 3 |
| `pl_slv_daily` | `layer = 'SLV' AND execution_order IN (2,3,4)` | `nb_slv_etl` | 4 |
| `pl_gld_daily` | `layer = 'GLD' AND execution_order = 5` | `nb_gld_etl` | 3 |

**Master pipeline pattern:**
```
pl_brz_daily (dependsOn: [])
  → pl_slv_daily (dependsOn: pl_brz_daily.Succeeded)
    → pl_gld_daily (dependsOn: pl_slv_daily.Succeeded)
```

---

### PHASE 3: METADATA TABLE EXTRACTION (CRITICAL)

This is the most important phase — reading `dbo.utl_pipeline_metadata` from the Delta Lake.

#### Step 6 — List Delta log files

```
Tool:    mcp_fabric_onelake
Command: onelake_list_files
Params:  {
  "workspace": "<WORKSPACE_ID>",
  "item": "<LAKEHOUSE_NAME>.Lakehouse",
  "path": "Tables/dbo.utl_pipeline_metadata/_delta_log",
  "recursive": true
}
```

Find the **latest** transaction log JSON file (highest numbered file, e.g., `00000000000000000139.json`).

#### Step 7 — Download the latest transaction log

```
Tool:    mcp_fabric_onelake
Command: onelake_download_file
Params:  {
  "workspace": "<WORKSPACE_ID>",
  "item": "<LAKEHOUSE_NAME>.Lakehouse",
  "file-path": "Tables/dbo.utl_pipeline_metadata/_delta_log/<LATEST_LOG_FILE>"
}
```

Parse the JSON to find:
- `add` actions → currently active parquet files
- `remove` actions → files to ignore

Build a list of **active parquet file paths**.

**IMPORTANT:** If the latest log is a checkpoint, you may need to also read prior logs. Walk backwards through the commit history if needed to reconstruct the full set of active files. Look for `.checkpoint.parquet` files too.

#### Step 8 — List the parquet data files

```
Tool:    mcp_fabric_onelake
Command: onelake_list_files
Params:  {
  "workspace": "<WORKSPACE_ID>",
  "item": "<LAKEHOUSE_NAME>.Lakehouse",
  "path": "Tables/dbo.utl_pipeline_metadata"
}
```

Cross-reference with the transaction log to determine which `.parquet` files are currently active.

#### Step 9 — Download active parquet files

For each active parquet file:

```
Tool:    mcp_fabric_onelake
Command: onelake_download_file
Params:  {
  "workspace": "<WORKSPACE_ID>",
  "item": "<LAKEHOUSE_NAME>.Lakehouse",
  "file-path": "Tables/dbo.utl_pipeline_metadata/<PARQUET_FILE>",
  "download-file-path": "/Users/MAC/Documents/MCP Fabric/<LOCAL_FILENAME>.parquet"
}
```

#### Step 10 — Parse parquet files with Python

Write and run a Python script to read the parquet files and extract metadata:

```python
import pyarrow.parquet as pq
import json, glob

files = glob.glob('/Users/MAC/Documents/MCP Fabric/*.parquet')
rows = []
for f in files:
    table = pq.read_table(f)
    df = table.to_pandas()
    for _, r in df.iterrows():
        rows.append({
            'table': str(r.get('table_name', '')),
            'layer': str(r.get('layer', '')),
            'order': int(r.get('execution_order', 0)),
            'load':  str(r.get('load_type', '')),
            'freq':  str(r.get('frequency', 'Daily')),
            'rows':  int(r.get('row_count', 0)) if r.get('row_count') else 0,
            'status': str(r.get('status', 'success')),
            'nb':    str(r.get('notebook_name', ''))[:12],
        })

# Filter active rows only
active = [r for r in rows if r['table']]
print(f"Total active tables: {len(active)}")
print(json.dumps(active, indent=2))
```

**Expected result:** 26 rows with this distribution:
- REF: 9 tables (order 1)
- BRZ: 7 tables (order 1)
- SLV: 8 tables (orders 2, 3, 4)
- GLD: 2 tables (order 5)

**VALIDATION:** Total must equal 26. If not, re-check delta log replay. Count per layer must match.

---

### PHASE 4: GENERATE VISUALIZATION

#### Step 11 — Build the DATA array

Transform the parsed metadata into the JavaScript `DATA` array format:

```javascript
const DATA = [
  {layer:'REF', order:1, table:'ref_calendar_2', load:'overwrite', freq:'Monthly', rows:21551, status:'success', nb:'1f372050-e7ac', runtime:'41s'},
  // ... one entry per table
];
```

Each entry MUST have these fields: `layer`, `order`, `table`, `load`, `freq`, `rows`, `status`, `nb`, `runtime`.

Optional field for SLV/GLD: `lineage` (array of upstream table name fragments).

#### Step 12 — Build the hierarchy object

The `hier` object defines the tree structure. Build it dynamically from the DATA:

```javascript
const ENGINES = {REF:'nb_etl', BRZ:'nb_etl', SLV:'nb_slv_etl', GLD:'nb_gld_etl'};

const hier = {
  id:'root', type:'master', name:'pl_master_daily',
  sub:'Sequential Orchestrator │ Schedule: Daily @ 10:00 AM', layer:'MASTER',
  children: [
    // BRZ pipeline with REF + BRZ groups
    {id:'pl_brz', type:'pipeline', layer:'BRZ', name:'pl_brz_daily', sub:'ForEach batchCount=3',
      children: [
        {id:'eng_ref', type:'engine', layer:'REF', name:'⚙ nb_etl', sub:'Bronze ETL Engine → all REF+BRZ tables call this notebook'},
        {id:'grp_ref', type:'group', layer:'REF', name:`REF · ${refCount} tables`, sub:'Monthly │ Overwrite',
          children: DATA.filter(d=>d.layer==='REF').map((d,i)=>({id:'r'+i, type:'table', layer:'REF', data:d}))},
        {id:'grp_brz', type:'group', layer:'BRZ', name:`BRZ · ${brzCount} tables`, sub:'Daily │ Overwrite/Incremental',
          children: DATA.filter(d=>d.layer==='BRZ').map((d,i)=>({id:'b'+i, type:'table', layer:'BRZ', data:d}))}
      ]},
    // SLV pipeline with stage groups per execution_order
    {id:'pl_slv', type:'pipeline', layer:'SLV', name:'pl_slv_daily', sub:'ForEach batchCount=4',
      children: [
        {id:'eng_slv', type:'engine', layer:'SLV', name:'⚙ nb_slv_etl', sub:'Silver ETL Engine → all SLV tables call this notebook'},
        // One group per distinct execution_order in SLV
        ...distinctOrders.map((ord,i) => ({
          id:'stg'+i, type:'group', layer:'SLV', name:`Stage ${i+1} · Order ${ord}`,
          sub:`${countForOrder} tables`,
          children: DATA.filter(d=>d.order===ord && d.layer==='SLV').map((d,j)=>({id:'s'+ord+'_'+j, type:'table', layer:'SLV', data:d}))
        }))
      ]},
    // GLD pipeline
    {id:'pl_gld', type:'pipeline', layer:'GLD', name:'pl_gld_daily', sub:'ForEach batchCount=3',
      children: [
        {id:'eng_gld', type:'engine', layer:'GLD', name:'⚙ nb_gld_etl', sub:'Gold ETL Engine → all GLD tables call this notebook'},
        ...DATA.filter(d=>d.layer==='GLD').map((d,i)=>({id:'g'+i, type:'table', layer:'GLD', data:d}))
      ]}
  ]
};
```

#### Step 13 — Generate the `index.html` / `pipeline_workflow.html`

Use the D3.js template at:
```
/Users/MAC/Documents/MCP Fabric/fabric_template_pipeline_decomposition/index.html
```

Copy this template to the output location. Then replace ONLY the `const DATA=[...]` block (lines 160–187 approximately) and the `const ENGINES={...}` line with the newly extracted data. Do NOT change any other part of the HTML/CSS/JS.

Also update:
- The `<title>` tag and `<h1>` with the actual workspace name
- The topbar subtitle with actual metadata source info
- The sidebar stats (total counts per layer, total rows, success rate)

#### Step 14 — Open in browser to verify

```bash
open pipeline_workflow.html
```

Use the browser tool to verify:
1. All nodes are visible in the tree
2. Total table count matches (should be 26 for SupplyChain-Dev)
3. No text overflows node boxes
4. Sidebar shows correct layer counts
5. Failed tables are highlighted in red

#### Step 15 — (Optional) Push to GitHub Pages

If the user requests GitHub deployment:

```bash
cd /Users/MAC/Documents/MCP Fabric/fabric_template_pipeline_decomposition
cp ../pipeline_workflow.html index.html
git add -A
git commit -m "🔄 Updated pipeline metadata ($(date +%Y-%m-%d))"
git push origin main
```

GitHub Pages URL: `https://ankinguyen-engineer-2002.github.io/fabric_template_pipeline_decomposition/`

---

## KEY DESIGN DECISIONS

### Node Types in the D3.js Tree

| Type | Visual | Width | Height |
|---|---|---|---|
| `master` | Blue glow, centered text | 210 | 48 |
| `pipeline` | Layer-colored border, accent bar | 150 | 38 |
| `engine` | Dashed border, layer-colored | 190 | 32 |
| `group` | Subtle border, accent bar | 150 | 34 |
| `table` | Solid dark fill, status dot | 185 | 30 |

### Fill Colors (MUST be fully opaque, not rgba)

```
master:   #0c1a3a
pipeline: computed solid from layer color at 12% blend with #060a14
engine:   #080e1e
group:    #0a1020
table:    #0d1525
failed:   #140a10
```

### Layer Colors

```javascript
const COL  = {BRZ:'#f97316', REF:'#06b6d4', SLV:'#a855f7', GLD:'#eab308'};
const COL2 = {BRZ:'#fb923c', REF:'#22d3ee', SLV:'#c084fc', GLD:'#facc15'};
```

### Text Truncation

All text MUST be truncated to fit within node boundaries:
- Table names: `maxChars = Math.floor((nodeWidth - 30) / 5)`
- Sub text: `maxChars = Math.floor((nodeWidth - 20) / 4.5)`
- Append `…` if truncated

### Sub-text color

Use `#8b9bb5` for readability (NOT `#64748b` which is too faint on dark backgrounds).

---

## TROUBLESHOOTING

| Issue | Solution |
|---|---|
| Parquet files return empty data | Check delta log — files may have been `remove`d. Replay from checkpoint. |
| Less than 26 tables | Filter by `is_active = 1` only. Some rows may be inactive. |
| Pipeline definition is base64 | Decode: `base64.b64decode(payload).decode('utf-8')` then `json.loads()` |
| Tree nodes overlap | Increase `nodeSize` in `d3.tree().nodeSize([vertical, horizontal])` |
| Text overflows boxes | Ensure truncation logic is applied in `.each()` callback |
| Lines show through nodes | All node `fill` must be solid hex colors (e.g., `#0d1525`), NOT `rgba()` |
| GitHub Pages 404 | Enable Pages from Settings → Pages → Source: main branch, / (root) |

---

## REFERENCE: utl_pipeline_metadata SCHEMA

| Column | Type | Description |
|---|---|---|
| `table_name` | string | Target Delta table name |
| `layer` | string | REF / BRZ / SLV / GLD |
| `execution_order` | int | Execution sequence (1→2→3→4→5) |
| `load_type` | string | overwrite / incremental |
| `notebook_name` | string | UUID of the ETL notebook |
| `is_active` | boolean | Whether to process this table |
| `frequency` | string | Daily / Monthly |
| `next_run_time` | timestamp | Next scheduled execution |
| `row_count` | bigint | Last known row count |
| `status` | string | success / failed |
