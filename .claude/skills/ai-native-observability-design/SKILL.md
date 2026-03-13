---
name: ai-native-observability-design
description: >
  Design the Observability module for an AI-Native Data Product on Teradata.
  Produces DDL for monitoring tables (change_event, data_quality_metric, data_lineage,
  model_performance, agent_outcome), standard monitoring queries, the closed-loop feed
  pattern into the Memory module, and Semantic module registration. Use this skill whenever
  the user asks to design or build an Observability module, implement audit trail or change
  tracking, define data quality metrics, capture data lineage (OpenLineage), monitor ML model
  performance, or track agent outcomes and feedback in an AI-Native data product.
---

# AI-Native Observability Module Design Skill

## Core Principle (Never Violate)

**Store events and metrics — never business data.**

| Store in Observability ✅ | Never store ❌ |
|--------------------------|---------------|
| "Party_H updated: 125,000 records by ETL_NIGHTLY" | Actual Party records |
| "Party_H completeness score: 0.97 at 2024-03-15" | Actual Party attributes |
| "Model churn_v2: AUC = 0.89, 50,000 samples" | Actual predictions (→ Prediction module) |
| "Agent QUERY action: SUCCESS, 342ms" | Query result sets |

**Scale**: Table-level change tracking only — one event per batch load, not one per record. A 10M-row ETL run produces one `change_event` row, not 10M.

---

## Role in the Architecture

Observability is built in **Phase 3** (after Domain, Semantic, Search, Prediction). It monitors everything and feeds learnings forward into Memory.

```
Domain / Search / Prediction  →  Observability (monitors health)
Observability                  →  Memory (feeds learned strategies)
```

**Observability monitors all six modules** — including itself.

---

## Design Workflow

### Step 1 — Gather Requirements

Confirm or ask for:
- Which tables/modules need change tracking?
- Which quality metrics to measure (completeness, validity, uniqueness, timeliness)?
- Quality thresholds per metric (e.g., completeness ≥ 0.95)
- ML models to monitor (which metrics: AUC, precision, drift score, latency)?
- Agent outcome tracking needed? (action types, feedback categories)
- OpenLineage integration in use? (if yes — namespace and job naming conventions)
- Retention policy per table (events accumulate — need time-based partitioning)

### Step 2 — Key Decisions

**Which tables to implement:**

| Table | Include When |
|-------|-------------|
| `change_event` | Always — foundational audit trail |
| `data_quality_metric` | Always — quality monitoring for all modules |
| `data_lineage` | ETL/ELT pipelines exist; OpenLineage integration desired |
| `model_performance` | Prediction module is deployed with ML models |
| `agent_outcome` | AI agents are actively using the data product |

**Partitioning strategy** (Observability tables grow continuously):

| Table Volume Expectation | Strategy |
|--------------------------|----------|
| Low event frequency | No partitioning initially |
| Daily ETL + quality runs | Partition by month on `change_dts` / `measured_dts` |
| High-frequency agent activity | Partition by week or day on `action_dts` |

**Retention policy** (define before go-live):
- `change_event`: typically 2–7 years (regulatory audit)
- `data_quality_metric`: 1–2 years (trend analysis)
- `data_lineage`: 1–3 years (lineage traceability)
- `model_performance`: lifetime of model + 1 year
- `agent_outcome`: 6–12 months rolling window

### Step 3 — Build in This Order

1. Core monitoring tables — DDL with full `COMMENT ON` statements
2. Standard monitoring views — cross-table summary views
3. Memory feed pattern — closed-loop learning queries
4. Register in Semantic module

### Step 4 — Documentation Capture

Read `../ai-native-documentation-design/references/documentation-capture.md` for the full protocol, SQL templates, and ID conventions.

**Module short name:** `OBSERVABILITY` — Decision IDs: `DD-OBSERVABILITY-{NNN}`, Change log: `CL-OBSERVABILITY-{NNN}`, Cookbook: `QC-OBSERVABILITY-{NNN}`

**Typical decisions to capture:**

| Decision | Category | Affects |
|----------|----------|---------|
| Tables implemented (which of the 5 core tables) | ARCHITECTURE | Module scope |
| Partitioning strategy (by month/week/day) | PERFORMANCE | All event tables |
| Retention policy per table | OPERATIONAL | All tables |
| Quality thresholds per metric | OPERATIONAL | `data_quality_metric` |
| OpenLineage scope (which pipelines, namespace) | INTEGRATION | `data_lineage` |
| Memory feed pattern (frequency, sample size) | INTEGRATION | `agent_outcome` → Memory |

**Typical glossary terms:** change event, data quality metric, data lineage, model performance, agent outcome, threshold, SLA, OpenLineage

**Typical cookbook entries:** quality failure detection query, change audit trail query, Memory feed query

**Required outputs:**
1. `dp_documentation.Module_Registry` INSERT (1 — module registration)
2. `dp_documentation.Design_Decision` INSERTs (minimum 3 — key architectural/schema choices)
3. `dp_documentation.Change_Log` INSERT (1 — CL-OBSERVABILITY-001, INITIAL_RELEASE v1.0.0)
4. `dp_documentation.Business_Glossary` INSERTs (minimum 3 — domain terms introduced)
5. `dp_documentation.Query_Cookbook` INSERTs (minimum 1 — key query patterns)

### Step 5 — Validate

Run the population test and integration checks in `references/checklists.md`.

---

## Boolean Conventions

All boolean flag columns follow Domain module standards:
- **Type**: `BYTEINT NOT NULL DEFAULT value`
- **Naming**: `is_` prefix
- **Values**: `1` = true/yes/pass, `0` = false/no/fail

Applied in Observability:
```
is_threshold_met   BYTEINT NOT NULL DEFAULT 0  -- data_quality_metric
is_sla_met         BYTEINT NOT NULL DEFAULT 0  -- model_performance
```

---

## Quality Metric Standard Names

Use these standard names in `metric_name` for cross-table consistency:

| `metric_name` | `metric_category` | Measures |
|--------------|------------------|----------|
| `COMPLETENESS` | `COMPLETENESS` | % non-null values |
| `UNIQUENESS` | `VALIDITY` | % distinct values |
| `VALIDITY` | `VALIDITY` | % passing format/range rules |
| `CONSISTENCY` | `CONSISTENCY` | % consistent across tables |
| `TIMELINESS` | `TIMELINESS` | % records within expected age |
| `ACCURACY` | `ACCURACY` | % matching reference source |

Compatible with Great Expectations, Deequ, Monte Carlo naming conventions.

---

## Open Standards Reference

- **OpenLineage**: https://openlineage.io — store `openlineage_run_id`, `openlineage_job_name`, `openlineage_namespace` in `data_lineage` for external catalog correlation (Marquez, Amundsen)
- **OpenTelemetry**: For performance metrics and distributed tracing correlation
- **Great Expectations / Deequ**: Use standard metric names above for framework compatibility

---

## Reference Files

| File | Read When |
|------|-----------|
| `references/observability-tables.md` | Writing DDL for all five core tables |
| `references/observability-queries.md` | Monitoring queries, Memory feed pattern, agent discovery |
| `references/checklists.md` | Final review, retention setup, Semantic registration |
| `../ai-native-documentation-design/references/documentation-capture.md` | Documentation Capture step — SQL templates and ID conventions |
