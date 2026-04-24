---
name: data-product-design
description: >
  Guide developers through designing well-structured, portable data products for lakehouse
  architectures. Use this skill proactively whenever a developer needs to design or document
  a new data product, define data models across medallion layers (bronze/silver/gold), add a
  transactional serving layer, specify on-demand metrics or KPIs with MetricFlow, design
  dataflow and job specs, write a data product contract, or think through catalog and semantic
  layer structure. Triggers on phrases like "design a data product", "what tables should I
  build", "how to structure my data product", "define metrics for a domain", "spec out a
  pipeline", "data product contract", "catalog and semantic layer", "medallion architecture
  for a domain", "on-demand simulations", "data model for supply chain / inventory / demand
  planning". Use it even when the user has not said "data product" explicitly — if they are
  asking how to model and expose domain data, this skill applies.
allowed-tools: Read, Write, Glob, Grep
---

# Data Product Design

## Purpose

This skill guides developers through designing a **complete, portable data product** — from raw ingestion to metrics to on-demand compute — producing a machine-readable spec package that works consistently across deployment tiers (Databricks lakehouse or lightweight open-source stacks).

A data product is not just tables. It is a **bounded, ownable unit** with four components that work together:

| Component | What it answers | Key artifact |
|---|---|---|
| **Physical layer** | What data exists and where | Medallion tables + transactional cache |
| **Semantic layer** | What the data means, how KPIs are computed | MetricFlow metric definitions |
| **Compute layer** | What on-demand jobs and simulations are available | Job + simulation specs |
| **Contract** | Who owns it, what it exposes, what the SLA is | `contract.yml` |

The catalog is the registry that makes all four discoverable. The semantic layer defines meaning. They are separate concerns.

---

## Architecture Overview

Every data product sits on top of **five layers**, each with a distinct role:

```
┌─────────────────────────────────────────────────────┐
│              CATALOG  (registry + governance)        │
├─────────────────────────────────────────────────────┤
│           SEMANTIC LAYER  (MetricFlow metrics)       │
├──────────────────────┬──────────────────────────────┤
│   GOLD (thin facts   │  TRANSACTIONAL LAYER         │
│   + dims, no KPIs)   │  (hot-path serving cache)    │
├──────────────────────┴──────────────────────────────┤
│              SILVER  (clean, conformed)              │
├─────────────────────────────────────────────────────┤
│              BRONZE  (raw, immutable)                │
└─────────────────────────────────────────────────────┘
```

### Bronze — Raw, Immutable
Exact copy of source data. Never modified after write. Append-only. Carries metadata columns: `_ingested_at`, `_source_system`, `_source_file`. The audit record and reprocessing safety net.

### Silver — Clean, Conformed
Deduplicated, validated, typed, business-rule-applied. The stable analytical foundation that downstream consumers trust. Enforces schema. No aggregation here — that is a semantic layer concern.

### Gold — Thin Facts and Dimensions
Conformed dimension tables and fact tables ready for joins. **Not a KPI or aggregation layer** — that job belongs to the semantic layer (MetricFlow). Keeping Gold lean is intentional: it reduces maintenance and prevents logic duplication between Gold tables and metric definitions.

> If you find yourself building a `gold_monthly_fill_rate` table, stop. Define `fill_rate` as a MetricFlow metric instead. The semantic layer computes it on-demand from Silver/Gold facts.

### Transactional Layer — Hot-Path Serving
A fast read projection of lakehouse data for app-facing queries that need sub-second latency. This is **not a source of truth** — the lakehouse always is. It is a materialized cache, kept in sync via CDC or scheduled sync jobs from Silver.

Use when:
- An app needs point-lookup by key (e.g., current stock level for SKU X at warehouse Y)
- Response time SLA < 200ms that Delta Lake cannot meet
- Active/current state queries (open orders, live replenishment plans)

Avoid when:
- The query is analytical (aggregations, trends, reports)
- Freshness tolerance is minutes or hours (Delta + Photon is fine)

### Semantic Layer — Metrics, Not Tables
MetricFlow YAML definitions. Each metric specifies its measure, dimensions, and filters once. Computed on-demand at query time by the MetricFlow engine against Silver/Gold tables. Exposed via SQL endpoints or the dbt Semantic Layer API depending on deployment tier.

**The catalog entry references what metrics exist. The MetricFlow YAML defines how they compute. These are separate.**

---

## Design Workflow

Work through these steps in order. Each step produces a spec artifact.

### Step 1 — Define the Domain and Boundary

Name the data product and its domain. A data product owns one bounded domain slice — resist the urge to make it a catch-all.

Ask:
- What business process or domain does this product serve?
- Who are the consumers? (analysts, app developers, ML engineers, planners)
- What questions must this product be able to answer?
- What does it explicitly **not** cover?

Document this in the `contract.yml` header. See `references/contract-template.yml` for the full schema.

### Step 2 — Design the Physical Model (Bronze → Silver → Gold)

For each layer, define the tables. Use Crow's Foot ERD notation for relationships (see `references/data-modeling-guide.md` for notation, normalization, type mapping, and data dictionary format). Apply the layer patterns in `references/layer-patterns.md` for incremental processing, quality checks, and optimization.

**Bronze tables** — one per source feed. Append-only, schema-flexible, metadata-enriched. No transformation or business logic. Read `references/layer-patterns.md § Bronze Layer Patterns`.

**Silver tables** — one per logical entity. Normalized (1NF → 3NF), typed, deduplicated, business-rule-applied. Define keys, relationships, and constraints using the normalization and physical model guidance in `references/data-modeling-guide.md`. Read `references/layer-patterns.md § Silver Layer Patterns` for transformation sequence, merge strategy, and quality gates.

**Gold tables** — conformed fact and dimension tables only. No aggregated KPI tables. Follow the star schema pattern. Define surrogate keys, SCD strategy if applicable, and partitioning strategy. Read `references/layer-patterns.md § Gold Layer Patterns`.

Output: a data model spec with ERD per layer and a table spec for each table (name, columns, types, keys, partitioning, refresh strategy). Use the format in `references/contract-template.yml` under `physical_layer`.

**Decision rule for Gold vs. Semantic layer:**

| It goes in Gold if... | It goes in Semantic Layer if... |
|---|---|
| It is a conformed fact or dimension | It involves aggregation, ratio, or formula |
| Multiple metrics join against it | It is a KPI or business metric |
| It needs a surrogate key for joins | It can be computed at query time in < 2s |

### Step 3 — Define the Transactional Layer

Identify which Silver entities need fast app-facing access. For each, specify:
- The read pattern (point lookup? range query? what key?)
- The sync mechanism (scheduled job, CDC, streaming — see `references/layer-patterns.md § Transactional Layer Patterns` for guidance)
- The freshness SLA
- The target store (Delta with cache, PostgreSQL, Redis — see the selection table in layer-patterns.md)

Output: transactional layer spec block in `contract.yml`. Read `references/layer-patterns.md § Transactional Layer Patterns` before designing this layer — in particular the rule about when not to use it.

### Step 4 — Define Metrics (Semantic Layer)

For each KPI or business metric the data product must expose, write a MetricFlow-style spec. These are YAML definitions that become the metric contract — independent of whether the execution target is Databricks Metric Views or dbt MetricFlow on DuckDB. See `references/layer-patterns.md § Semantic Layer Patterns` for metric types, dimension handling, and design rules.

```yaml
metrics:
  - name: fill_rate
    description: "Ratio of orders fulfilled to total orders in period"
    type: ratio
    numerator:
      measure: orders_fulfilled
      filter: "status = 'FULFILLED'"
    denominator:
      measure: total_orders
    dimensions:
      - region
      - warehouse_id
      - product_category
      - order_date        # time dimension — supports grain: day/week/month
    source_table: silver.fct_orders

  - name: inventory_turnover
    description: "COGS divided by average inventory value"
    type: derived
    expr: "cogs / ((opening_inventory + closing_inventory) / 2)"
    dimensions:
      - sku_id
      - warehouse_id
      - fiscal_month
    source_tables:
      - silver.fct_inventory_movements
      - silver.dim_products
```

Each metric definition must include: name, description, type (simple/ratio/derived/cumulative), measure(s), dimensions, source table(s), and any filters.

### Step 5 — Define Jobs and Dataflows (Specs, Not Code)

For every data movement and transformation in the product, write a job spec. These are the contracts your orchestration layer (Lakeflow, Prefect, Dagster, cron) will implement — but they are written before any code to make the data flows reviewable and version-controlled.

Read `references/job-spec-template.yml` for the full format. Key fields per job:

- `name` and `type` (ingestion / transformation / aggregation / simulation / sync)
- `trigger` (scheduled / event / on_demand / streaming)
- `inputs` (sources with format and location)
- `outputs` (target layer and table)
- `sla` (max duration, freshness tolerance)
- `quality_checks` (what must pass before the job is considered successful)
- `dependencies` (upstream jobs this depends on)

**Dataflow diagram** — after defining all jobs, draw the lineage:

```
[Source: ERP] ──ingestion──► [Bronze: demand_raw]
                                      │
                              ──transform──► [Silver: fct_demand]
                                                    │
                              ──gold_load──► [Gold: fct_demand_conformed]
                                                    │
                              ◄──semantic──  [Metric: forecast_accuracy]
```

This diagram goes in the contract as `dataflow.lineage`.

### Step 6 — Define Simulations (On-Demand Compute Specs)

Simulations are parameterized jobs that run on-demand and write to scenario-tagged output tables. They are neither medallion layer jobs nor semantic layer definitions — they are first-class compute artifacts registered in the catalog.

For each simulation, define:
- Input parameters (name, type, required, default)
- Input data tables consumed
- Output table(s) written (always include `scenario_id` and `run_timestamp` partitioning)
- Estimated compute duration and tier
- Trigger mechanism

```yaml
simulations:
  - name: reorder_point_sim
    description: "Recalculates reorder points given variable supplier lead times"
    parameters:
      - name: supplier_id
        type: string
        required: true
      - name: lead_time_delta_days
        type: integer
        default: 0
      - name: service_level_target
        type: float
        default: 0.95
    inputs:
      - table: silver.fct_inventory_positions
      - table: silver.fct_demand_history
      - table: gold.dim_suppliers
    outputs:
      - table: simulations.reorder_point_results
        partitioned_by: [scenario_id, run_timestamp]
    compute:
      type: serverless_job
      estimated_duration_minutes: 5
    trigger: on_demand
```

### Step 7 — Write the Contract

The `contract.yml` is the data product's public interface and catalog registration document. It references all artifacts defined in steps 1–6. Read `references/contract-template.yml` for the full schema.

The contract answers:
- What is this product and who owns it?
- What tables does it expose and at what SLA?
- What metrics does it expose?
- What simulations can be triggered?
- What jobs keep it alive?
- What are the access policies?

---

## Portability Principle

Every spec in this skill is deployment-tier agnostic. The same contract and metric definitions deploy to:

| Tier | Table format | Catalog | Metrics execution | Compute |
|---|---|---|---|---|
| **Premium** (Databricks) | Delta Lake | Unity Catalog | MetricFlow → Databricks SQL Warehouse | Lakeflow Serverless |
| **Standard** (open source) | Apache Iceberg | Apache Polaris / Nessie | MetricFlow → Trino / Spark | Dagster / Prefect |
| **Lightweight** | Iceberg / DuckDB | Project Nessie | MetricFlow → DuckDB | Prefect / cron |

The catalog in all tiers implements the **Iceberg REST Catalog API** — the same interface your apps use to discover data products regardless of what's running behind it.

When designing, never let a platform-specific feature drive the spec. If a design only works on Databricks, it is not a portable data product — it is a Databricks pipeline.

---

## Output Artifact Checklist

A complete data product design produces:

- [ ] `contract.yml` — full data product contract (use template in references/)
- [ ] `data_model.md` — ERD and table specs per layer (Bronze, Silver, Gold)
- [ ] `metrics.yml` — MetricFlow metric definitions for all KPIs
- [ ] `jobs/` — one job spec YAML per dataflow job
- [ ] `simulations/` — one spec YAML per on-demand simulation
- [ ] `dataflow.md` — lineage diagram connecting all jobs and tables

---

## Reference Files

All reference files are bundled within this skill — no external dependencies.

| File | Contents | Read when |
|---|---|---|
| `references/contract-template.yml` | Fully annotated `contract.yml` template with supply chain examples | Step 1 and Step 7 |
| `references/job-spec-template.yml` | Job and simulation spec templates (ingestion, transformation, sync, simulation) | Step 5 and Step 6 |
| `references/data-modeling-guide.md` | ERD notation (Crow's Foot + Mermaid), normalization (1NF–3NF), SCD strategies, physical type mapping, data dictionary format | Step 2 |
| `references/layer-patterns.md` | Bronze/Silver/Gold/Transactional/Semantic layer patterns, incremental processing, quality checks per layer, optimization strategies, anti-patterns | Steps 2–4 |
