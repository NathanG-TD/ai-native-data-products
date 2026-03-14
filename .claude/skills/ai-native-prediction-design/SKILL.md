---
name: ai-native-prediction-design
description: >
  Design the Prediction module (Feature Store) for an AI-Native Data Product on Teradata.
  Produces DDL for feature tables (wide and tall format), model prediction storage, standard
  views for current features and point-in-time reconstruction, and Semantic module registration.
  Use this skill whenever the user asks to design or build a Prediction module, define a feature
  store, create engineered ML features, implement point-in-time correct training datasets,
  store model predictions, or integrate ML features with Domain entities in an AI-Native data product.
---

# AI-Native Prediction Module Design Skill

## Core Principle (Never Violate)

**Store ENGINEERED features only. Never copy raw Domain values.**

| Store in Prediction ✅ | Keep in Domain (use views) ❌ |
|----------------------|------------------------------|
| `recency_score` = days normalised to 0–1 | `last_transaction_dts` (raw) |
| `income_normalised` = income / max_income | `annual_income_amt` (raw) |
| `transaction_velocity` = count / max_count | `transaction_count` (raw) |
| `credit_utilisation` = balance / limit | `credit_limit_amt` (raw) |

**Exception**: Raw domain values may be duplicated only for **low-latency scoring** (< 100ms requirement where Domain joins would exceed latency budget). Must be documented per column.

---

## Role in the Architecture

Prediction is built in **Phase 2** (alongside Search, after Domain and Semantic).

```
Domain      → Source entities and raw values
Prediction  → Engineered features + model outputs
Semantic    → Feature definitions and computation logic (metadata only)
Observability → Feature drift and quality monitoring
```

**Key separation**: Semantic stores *what* a feature means and *how* it's computed. Prediction stores *the actual values*.

---

## Design Workflow

### Step 1 — Gather Requirements

Confirm or ask for:
- ML use cases (churn prediction, fraud scoring, product affinity, …)
- Entities to feature-engineer (Party, Product, Transaction, …)
- Feature groups and individual features per group
- Feature computation logic (normalisation formula, aggregation window, …)
- Dominant access pattern: features accessed together or individually?
- Expected feature set size: fixed and dense, or evolving and sparse?
- Real-time scoring latency requirement (drives low-latency duplication decision)
- Feature refresh frequency (real-time, daily batch, on-demand)
- Training data retention policy

### Step 2 — Key Decisions

**Storage pattern per feature group:**

| Condition | Pattern |
|-----------|---------|
| Features always accessed together, fixed set, dense | **Wide format** (one row per entity per snapshot) |
| Features accessed individually, evolving set, sparse | **Tall format** (one row per feature per entity) |
| Mixed needs | Both patterns in same product — designer decides per group |

**Entity FK pattern:**

| Condition | Choice |
|-----------|--------|
| Features apply to one known entity type | Specific FK: `party_key BIGINT` |
| Features apply to multiple entity types | Generic FK: `entity_key BIGINT + entity_type VARCHAR(50)` |

**Normalisation scale** (document per feature group):
- `[0, 1]` — most common, use `(value - min) / (max - min)`
- `[-1, 1]` — when negative values meaningful
- `z-score` — when normal distribution assumed: `(value - mean) / std_dev`

### Step 3 — Build in This Order

1. Feature group tables — wide or tall DDL with full `COMMENT ON` statements
2. `model_prediction` table — if model outputs need to be stored
3. Standard views — `_current`, `_enriched`, `_pit` for each feature group
4. Point-in-time query patterns — parameterised for training pipelines
5. Register in Semantic module — feature definitions in `column_metadata`

### Step 4 — Documentation Capture

Read `../ai-native-documentation-design/references/documentation-capture.md` for the full protocol, SQL templates, ID conventions, and **output file convention**.

**Output file**: Write documentation SQL as the last numbered file in the Prediction module directory: `03-prediction/{NN}-documentation.sql` (e.g., `03-prediction/03-documentation.sql` if 02 is the last existing file). Include a Deploy comment in the header (e.g., `-- Deploy: Phase 2, after Prediction DDL`).

**Module short name:** `PREDICTION` — Decision IDs: `DD-PREDICTION-{NNN}`, Change log: `CL-PREDICTION-{NNN}`, Cookbook: `QC-PREDICTION-{NNN}`

**Typical decisions to capture:**

| Decision | Category | Affects |
|----------|----------|---------|
| Storage pattern per feature group (wide vs tall) | ARCHITECTURE | Feature tables |
| Normalisation approach (min-max, z-score, etc.) | SCHEMA | Feature columns |
| Entity FK pattern (specific vs generic) | INTEGRATION | All feature tables |
| Refresh strategy (real-time, daily batch, on-demand) | OPERATIONAL | All feature tables |
| Model prediction format and storage | SCHEMA | `model_prediction` |
| Low-latency duplication exceptions (if any) | PERFORMANCE | Specific columns |

**Typical glossary terms:** engineered feature, feature group, observation timestamp, point-in-time, feature drift, normalisation scale, model prediction

**Typical cookbook entries:** point-in-time feature reconstruction query, current feature retrieval, training dataset generation

**Required outputs:**
1. `dp_documentation.Module_Registry` INSERT (1 — module registration)
2. `dp_documentation.Design_Decision` INSERTs (minimum 3 — key architectural/schema choices)
3. `dp_documentation.Change_Log` INSERT (1 — CL-PREDICTION-001, INITIAL_RELEASE v1.0.0)
4. `dp_documentation.Business_Glossary` INSERTs (minimum 3 — domain terms introduced)
5. `dp_documentation.Query_Cookbook` INSERTs (minimum 1 — key query patterns)

### Step 5 — Validate

Run the point-in-time correctness test and integration checks in `references/checklists.md`.

---

## Quick Reference

**Required temporal columns (every feature table):**
```
observation_dts   TIMESTAMP(6) WITH TIME ZONE NOT NULL  -- when feature was observed/computed
valid_from_dts    TIMESTAMP(6) WITH TIME ZONE NOT NULL  -- when this version became valid
valid_to_dts      TIMESTAMP(6) WITH TIME ZONE NOT NULL  -- DEFAULT '9999-12-31 23:59:59.999999+00:00'
is_current        BYTEINT NOT NULL DEFAULT 1            -- 1 = current, 0 = historical
```

**Standard filter for current features:**
```sql
WHERE is_current = 1   -- BYTEINT, consistent with Domain module
```

**Standard view naming:**
```
v_{entity}_features_current    -- Current engineered feature values only
v_{entity}_features_enriched   -- Features + Domain context (JOIN, no duplication)
v_{entity}_features_pit        -- Point-in-time reconstruction for training
```

**Feature engineering formulas:**
```
Min-max normalisation:  (value - min) / (max - min)  → [0, 1]
Max normalisation:      value / MAX(value)            → [0, 1]  (simpler, common)
Z-score:                (value - mean) / std_dev      → centred
Recency:                1.0 - (days_elapsed / window) → recent = higher
Utilisation:            numerator / denominator       → ratio [0, 1]
```

**Open standards reference:**
- Feast (open-source feature store): https://feast.dev/
- Tecton: feature engineering best practices
- MLOps patterns: versioning, lineage, drift monitoring

---

## Reference Files

| File | Read When |
|------|-----------|
| `references/prediction-tables.md` | Writing DDL for feature tables, model prediction table, views |
| `references/prediction-queries.md` | Point-in-time queries, training dataset patterns, agent discovery |
| `references/checklists.md` | Final review, Semantic registration, validation tests |
| `../ai-native-documentation-design/references/documentation-capture.md` | Documentation Capture step — SQL templates and ID conventions |
