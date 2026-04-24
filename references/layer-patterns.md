# Layer Patterns

Implementation patterns for each architectural layer: Bronze, Silver, Gold, Transactional,
and Semantic. Covers incremental processing, quality checks, optimization, and anti-patterns.
These are spec-level patterns — orchestration and code are deployment-specific.

---

## Bronze Layer Patterns

### Purpose
Immutable historical record of source data exactly as received. Bronze is the safety net —
it enables reprocessing, debugging, and audit without going back to the source system.

### Characteristics
- **Append-only**: Never update or delete Bronze records
- **Schema-flexible**: Use schema-on-read or merge-schema to accommodate upstream drift
- **Metadata-enriched**: Every row carries system columns added at ingestion time
- **No business logic**: Transformations belong in Silver, not Bronze

### Required Metadata Columns

| Column | Type | Value |
|---|---|---|
| `_ingested_at` | TIMESTAMP | Ingestion timestamp (system clock) |
| `_source_system` | STRING | Source system identifier (e.g., `'erp'`, `'wms'`) |
| `_source_file` | STRING | Source file path or batch ID (for file-based ingestion) |
| `_ingestion_date` | DATE | Derived from `_ingested_at` — used as partition column |

### Partitioning
Always partition Bronze by `_ingestion_date`. This makes time-range pruning and reprocessing
efficient without depending on business date columns that may be missing or malformed.

### Quality Checks (Bronze — Minimal)
Bronze checks should be lightweight and non-blocking. The goal is to confirm the feed arrived
and is not empty, not to validate business rules.

- File/stream received (non-zero row count)
- Schema not regressed (column count not lower than prior run)
- Row count within expected range (alert on extreme deviation, do not block)

Do **not** enforce null checks, type validation, or referential integrity in Bronze — that is Silver's job.

### Incremental Pattern
For file-based ingestion, use Auto Loader (Databricks) or a file-manifest approach (open source)
to process only new files since the last run. Track the watermark as the last `_ingested_at`
value processed.

For CDC / streaming sources, append events to Bronze using a streaming write. Bronze becomes the
event log. Silver applies the merge/upsert logic.

### Anti-Patterns
- ❌ Filtering rows in Bronze ("skip bad records") — you lose the audit trail
- ❌ Applying transformations in Bronze — makes reprocessing unpredictable
- ❌ Skipping Bronze entirely to "save storage" — you lose the ability to reprocess without re-extracting from source

---

## Silver Layer Patterns

### Purpose
The stable, trusted analytical foundation. Silver answers: "what is the clean, conformed version
of this entity?" It is the source of truth for downstream metrics and Gold tables.

### Characteristics
- **Schema-enforced**: Explicit schema, type-safe columns
- **Deduplicated**: One row per logical entity per state
- **Business-rule-applied**: Domain rules enforced (valid statuses, positive quantities, etc.)
- **Relationship-aware**: Foreign keys to conformed dimensions

### Transformation Sequence
Apply transformations in this order to avoid cascading issues:

1. **Filter nulls on mandatory keys** — drop records with null primary or business keys
2. **Cast types** — convert strings to dates, integers, decimals as defined in physical model
3. **Standardize values** — normalize status codes, country codes, UoM to canonical forms
4. **Deduplicate** — keep latest record per business key (use `_ingested_at` or source timestamp as tiebreaker)
5. **Validate business rules** — apply range checks, referential integrity, allowed-value checks
6. **Enrich** — join to reference/dimension tables to add denormalized lookup columns
7. **Add audit columns** — `_processed_at`, `_data_product_version`

### Merge Strategy (Incremental)
Silver tables are maintained as current-state tables via upsert (MERGE). Process only records
with `_ingested_at` newer than the last Silver watermark.

```
MERGE INTO silver.fct_demand AS target
USING new_records AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
```

For SCD2 dimensions, do not overwrite — insert a new row with updated `valid_from` / `valid_to`
and set `is_current = false` on the prior row.

### Quality Gates
Silver quality checks are **blocking** — a failed check should halt the job and not propagate
bad data downstream.

| Check | Applies to |
|---|---|
| No nulls on primary key | Every Silver table |
| No nulls on required foreign keys | Every Silver table with FK relationships |
| No duplicate primary keys | Every Silver table |
| Type conformance | All typed columns |
| Allowed values | STATUS, TYPE, CATEGORY columns |
| Positive values | Quantity, amount, duration columns |
| Date sanity | Dates not in the future (where applicable); dates not before system inception |
| Referential integrity | FK columns — verify referenced rows exist in dimension tables |
| Row count delta | Alert if row count change vs. prior run exceeds ±20% |

### Change Data Feed
Enable Change Data Feed (CDF) on Silver Delta tables. This allows Gold jobs and the transactional
sync to consume only changed rows efficiently, rather than full-table scans.

```sql
ALTER TABLE silver.fct_demand
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

On Iceberg: use incremental reads via snapshot diff API.

---

## Gold Layer Patterns

### Purpose
Conformed, query-ready dimensional model. Gold contains the facts and dimensions that metrics
are computed against. It does **not** contain pre-aggregated KPI tables — those belong in the
semantic layer.

### What Belongs in Gold

| Belongs in Gold | Does NOT belong in Gold |
|---|---|
| Conformed fact tables | Pre-aggregated KPI tables |
| Conformed dimension tables | Metric calculations |
| Surrogate keys for joins | Business logic that could be a MetricFlow metric |
| SCD2 dimension history | Redundant copies of Silver data |

### Conformed Dimensions
A conformed dimension is shared across multiple fact tables and data products. Examples:
`dim_products`, `dim_warehouses`, `dim_suppliers`, `dim_calendar`. These should be owned by
a single data product (e.g., a `master_data` product) and referenced by domain products via FK.

If your domain data product needs a dimension that doesn't exist yet, either:
- Create it in your product as a local dimension (with a note that it should eventually be promoted)
- Coordinate with the team that should own it

### Star Schema Structure
Gold tables follow a star schema: one central fact table surrounded by dimension tables.

```
dim_calendar ──┐
dim_products ──┤
               ├── fct_demand (central fact)
dim_warehouses─┤
dim_customers──┘
```

Fact tables contain: foreign keys to dimensions + measures (quantities, amounts, durations).
Dimension tables contain: descriptive attributes about an entity.

### Optimization

**Partitioning**: Partition Gold fact tables on the primary date dimension (e.g., `order_date`,
`transaction_date`). This enables efficient time-range queries which are the most common pattern
in supply chain analytics.

**Clustering / Z-ORDER**: Apply on the columns most frequently combined in filters:
```sql
-- Optimize Gold fact on common filter pattern
OPTIMIZE gold.fct_demand ZORDER BY (sku_id, warehouse_id);

-- Or use liquid clustering (Delta 3.x+) for auto-adaptive clustering
CREATE TABLE gold.fct_demand
CLUSTER BY (sku_id, warehouse_id, order_date);
```

**Vacuum**: Run VACUUM regularly (retain 7+ days for time travel). Automate as part of the
maintenance job spec.

### Refresh Strategy
Gold tables are rebuilt from Silver, not from Bronze. Use CDF on Silver tables to process only
changed rows since the last Gold run. For slowly changing dimensions, apply SCD logic here.

Full refresh is acceptable for small dimension tables (< 10M rows). Use incremental merge for
large fact tables.

---

## Transactional Layer Patterns

### Purpose
A fast read projection of lakehouse data for app-facing queries that need sub-second latency.
The lakehouse (Silver/Gold) is always the source of truth. The transactional layer is a
materialized cache — it can be rebuilt from Silver at any time.

### When to Use It

Use the transactional layer when all three conditions are true:
1. An application (not an analyst) needs the data
2. The query pattern is a point lookup or narrow range query (not aggregation)
3. The latency SLA cannot be met by Delta/SQL Warehouse (typically: < 200ms required)

### Common Supply Chain Use Cases
- Current stock level per SKU per warehouse (ATP queries)
- Open order status by order ID
- Active replenishment plan for a supplier
- Current safety stock and reorder point per SKU

### Sync Patterns

**Scheduled sync** (most common): A job runs on a schedule (e.g., every 15 minutes), reads
changed rows from Silver via CDF, and writes them to the serving store via upsert.

**Event-driven sync**: Triggered by job completion events. Lower latency than scheduled, more
complex to orchestrate.

**Streaming CDC**: Continuous propagation of Silver changes to the serving store. Use only when
freshness SLA < 5 minutes and operational complexity is justified.

### Target Store Selection

| Condition | Recommended store |
|---|---|
| Point lookup, < 200ms, moderate volume | Delta with Photon row cache |
| Point lookup, < 50ms, high read throughput | Redis or DynamoDB |
| Range queries, relational joins, < 200ms | PostgreSQL with indexes |
| Lightweight deployment, low volume | DuckDB in-process |

### Data Integrity Rule
The transactional layer must never be the write path for source-of-record data. If an app
needs to write (e.g., a planner confirms a replenishment order), the write goes to the
operational source system or directly to Bronze — not to the transactional cache.

---

## Semantic Layer Patterns

### Purpose
Defines what data means in business terms and how KPIs are computed. The semantic layer
sits above the physical tables and translates facts and dimensions into metrics, dimensions,
and entities that consumers can query without knowing the underlying table structure.

### MetricFlow Spec Format
All metrics are defined as YAML. The definition is the contract — how it executes depends
on the deployment tier (Databricks Metric Views, dbt MetricFlow → Databricks SQL Warehouse,
dbt MetricFlow → DuckDB, etc.).

### Metric Types

| Type | Description | Example |
|---|---|---|
| `simple` | Single aggregation of one measure | `total_orders = COUNT(order_id)` |
| `ratio` | Numerator measure / denominator measure | `fill_rate = fulfilled / total` |
| `derived` | Formula over other metrics or measures | `inventory_turnover = cogs / avg_inventory` |
| `cumulative` | Running total or rolling window | `rolling_90d_demand = SUM(qty) OVER 90 days` |

### Dimension Handling
Every metric must declare its valid dimensions — the attributes by which it can be sliced.
Dimensions come from the dimension tables the metric's source table joins to.

```yaml
metrics:
  - name: fill_rate
    type: ratio
    numerator:
      measure: orders_fulfilled        # COUNT or SUM with filter
      filter: "status = 'FULFILLED'"
    denominator:
      measure: total_orders            # COUNT all orders
    source_table: silver.fct_demand
    dimensions:
      - region                         # from dim_warehouses
      - warehouse_id                   # from dim_warehouses
      - product_category               # from dim_products
      - order_date                     # time dimension
    time_grains: [day, week, month, quarter]
```

### Rules for Metric Design
- Every metric should have a plain-language `description` field — this is what developers read in the catalog
- Never define the same metric twice under different names — harmonize early
- If a metric requires a join to compute, specify both `source_table` and `join_tables`
- For ratio metrics, always handle division-by-zero explicitly in the denominator filter
- Document what the metric **excludes** (e.g., "excludes cancelled orders") — omissions cause the most confusion

### What Does NOT Go in the Semantic Layer
- Raw column access (just query the table directly)
- Data quality metrics (those are pipeline metadata, not business metrics)
- Simulation outputs (those are compute artifacts, not semantic definitions)

---

## Quality Check Reference

Summary of quality checks by layer for use in job specs.

### Bronze
| Check | Blocking? | Notes |
|---|---|---|
| File received / rows > 0 | Yes | Alert immediately if feed is missing |
| Schema not regressed | Warning | Alert but do not block on schema addition |
| Row count within range | Warning | Alert on ±50% deviation from baseline |

### Silver
| Check | Blocking? | Notes |
|---|---|---|
| No null primary keys | Yes | Hard block |
| No duplicate primary keys | Yes | Hard block |
| No null required FKs | Yes | Hard block |
| Type conformance | Yes | Hard block |
| Referential integrity | Yes | Hard block |
| Allowed values (enums) | Yes | Hard block |
| Positive values (quantities) | Yes | Hard block |
| Date sanity | Yes | Hard block |
| Row count delta ±20% | Warning | Alert, do not block |

### Gold
| Check | Blocking? | Notes |
|---|---|---|
| No null surrogate keys | Yes | Hard block |
| Row count not decreased unexpectedly | Yes | Block if > 10% row loss |
| SCD2 no overlapping validity periods | Yes | Hard block |
| Aggregate totals reconcile to Silver | Warning | Alert on > 0.1% variance |

### Transactional
| Check | Blocking? | Notes |
|---|---|---|
| Serving store reachable | Yes | Block and alert |
| Row count non-zero | Yes | Block — empty cache is worse than stale cache |
| Last sync within freshness SLA | Warning | Alert to on-call |
