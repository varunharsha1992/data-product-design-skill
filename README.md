# Data Product Design Skill

Design complete, portable data products for lakehouse architectures with a consistent structure across medallion layers, semantic metrics, transactional serving, jobs, and contracts.

## What this skill helps with

Use this skill when you need to:
- Design a new domain data product end-to-end
- Define Bronze, Silver, and Gold table boundaries
- Add a transactional serving layer for low-latency reads
- Specify KPI/metric definitions with MetricFlow
- Write dataflow job specs and simulation specs
- Produce a machine-readable `contract.yml`

## Core model

The skill treats a data product as four coordinated components:
- Physical layer (tables and storage layout)
- Semantic layer (metrics and KPI logic)
- Compute layer (jobs and simulations)
- Contract layer (ownership, SLAs, and exposed interfaces)

It also uses a catalog as the discoverability and governance registry.

## Design workflow

The workflow in `SKILL.md` is organized into seven steps:
1. Define domain and product boundary
2. Design Bronze/Silver/Gold physical model
3. Define transactional serving layer
4. Define semantic metrics (MetricFlow style)
5. Specify jobs and lineage/dataflow
6. Define on-demand simulations
7. Write and finalize `contract.yml`

### Guidance by step

- **Step 2:** Use `references/data-modeling-guide.md` for Crow's Foot ERD notation, normalization (1NF-3NF), type mapping, and data dictionary format.
- **Steps 2-4:** Use `references/layer-patterns.md` for layer-specific patterns, quality gates, incremental processing, optimization, and anti-patterns.

## Expected output artifacts

A complete design should produce:
- `contract.yml`
- `data_model.md`
- `metrics.yml`
- `jobs/` job spec YAML files
- `simulations/` simulation spec YAML files
- `dataflow.md`

## Portability principle

The same logical design should stay deployment-tier agnostic. Specs are intended to work across:
- Databricks-centric lakehouse stacks
- Open-source Iceberg-based stacks
- Lightweight local/embedded stacks

If a design only works on one vendor platform, it is considered non-portable.

## References in this skill

- `references/contract-template.yml`
- `references/job-spec-template.yml`
- `references/data-modeling-guide.md`
- `references/layer-patterns.md`

For complete guidance and detailed examples, use `SKILL.md` as the source of truth.
