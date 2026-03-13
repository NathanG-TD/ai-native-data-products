# Design Review Checklists
## AI-Native Prediction Module

---

## 1. Designer Requirements Checklist

Before writing DDL, confirm:

- [ ] ML use cases identified (churn, fraud, affinity, …)
- [ ] Entities to feature-engineer identified
- [ ] Feature groups and individual features per group defined
- [ ] Computation logic documented per feature (formula, source columns, normalisation method)
- [ ] Storage pattern chosen per group: wide vs tall (and rationale documented)
- [ ] Entity FK pattern chosen: specific (`{entity}_key`) vs generic (`entity_key + entity_type`)
- [ ] Low-latency scoring requirements identified (determines if any raw Domain duplication acceptable)
- [ ] Feature refresh frequency documented (real-time, daily, on-demand)
- [ ] Training data retention policy defined
- [ ] Model prediction storage needed? (if yes — which models, output formats)

---

## 2. Structural Checklist

For each feature table:

- [ ] `feature_group_key` (wide) or `feature_value_key` (tall) — IDENTITY surrogate key
- [ ] Entity FK present (`entity_key + entity_type` OR `{entity}_key`)
- [ ] **No raw Domain values** — only engineered/transformed features (run anti-pattern check below)
- [ ] All engineered features have `COMMENT ON COLUMN` documenting: what it measures, formula, normalisation scale
- [ ] `observation_dts` — when feature was computed (NOT NULL)
- [ ] `valid_from_dts` / `valid_to_dts` — temporal versioning (NOT NULL)
- [ ] `is_current` — `BYTEINT NOT NULL DEFAULT 1` (consistent with Domain module)
- [ ] `feature_group_version` — versioning for A/B testing and rollback
- [ ] `computation_dts` and `source_system` — lineage tracking
- [ ] `COMMENT ON TABLE` — describes what group, engineered nature, access pattern
- [ ] `PRIMARY INDEX` — on surrogate key (NUPI for temporal tables)

For model prediction table (if used):

- [ ] `model_id` + `model_version` — both required for reproducibility
- [ ] `feature_observation_dts` — links prediction to its input feature state
- [ ] `confidence_score` — model confidence included
- [ ] `is_current = BYTEINT NOT NULL DEFAULT 1`

---

## 3. Anti-Pattern Audit — Raw Domain Value Check

Run this to catch any columns that look like direct Domain copies:

```sql
-- Check Prediction tables for suspicious column names
-- (columns that suggest raw domain values rather than engineered features)
SELECT TableName, ColumnName, ColumnType
FROM DBC.ColumnsV
WHERE DatabaseName = '{PredictionDatabase}'
  AND (
    -- Common raw domain column patterns — these should NOT appear in Prediction tables
    ColumnName LIKE '%_amt'         -- raw monetary amounts (use normalised versions)
    OR ColumnName LIKE '%_date'     -- raw dates (derive age/recency instead)
    OR ColumnName IN ('legal_name', 'birth_date', 'email_address',
                      'phone_number', 'credit_limit_amt', 'annual_income_amt')
  )
ORDER BY TableName, ColumnName;
-- Review each result — confirm either it's engineered or low-latency duplication is documented
```

**Any result requires a decision:**
- Is it genuinely engineered? → Verify `COMMENT ON COLUMN` documents the formula
- Is it raw domain value? → Move to Domain and access via `v_{entity}_features_enriched` view
- Is it low-latency exception? → Document rationale in `COMMENT ON COLUMN`

---

## 4. Views Checklist

- [ ] `v_{entity}_features_current` — filters `is_current = 1`, engineered features only
- [ ] `v_{entity}_features_enriched` — joins to Domain for raw context, no duplication
- [ ] `v_{entity}_features_pit` — all versions, no `is_current` filter, for training
- [ ] All views have `COMMENT ON VIEW` explaining purpose and when to use
- [ ] `_enriched` view uses `Domain.{Entity}_H` with `is_current = 1 AND is_deleted = 0`

---

## 5. Point-in-Time Correctness Test

```sql
-- Test: Features at training_date should NOT include information from after that date

-- Step 1: Pick a known training date
-- DECLARE training_date TIMESTAMP = TIMESTAMP '2024-03-01 00:00:00+00:00';

-- Step 2: Retrieve features using point-in-time pattern
SELECT entity_key, {feature_columns}, observation_dts
FROM Prediction.{entity}_{group}_features
WHERE entity_key    = {test_entity_key}
  AND observation_dts <= TIMESTAMP '2024-03-01 23:59:59.999999+00:00'
  AND valid_from_dts  <= TIMESTAMP '2024-03-01 23:59:59.999999+00:00'
  AND valid_to_dts    >  TIMESTAMP '2024-03-01 00:00:00+00:00';
-- Expected: returns feature snapshot valid on 2024-03-01

-- Step 3: Verify no future observation_dts returned
SELECT COUNT(*) AS future_leakage_cnt
FROM Prediction.{entity}_{group}_features
WHERE entity_key    = {test_entity_key}
  AND observation_dts > TIMESTAMP '2024-03-01 23:59:59.999999+00:00'
  AND valid_from_dts  <= TIMESTAMP '2024-03-01 23:59:59.999999+00:00'
  AND valid_to_dts    >  TIMESTAMP '2024-03-01 00:00:00+00:00';
-- Expected: 0 (no future data leaking into training window)
```

---

## 6. Semantic Module Registration Checklist

After building the Prediction module, register in Semantic:

**entity_metadata** — one row per feature table and model prediction table:
```sql
INSERT INTO Semantic.entity_metadata
(entity_name, entity_description, module_name, database_name,
 table_name, view_name, surrogate_key_column, natural_key_column,
 temporal_pattern, current_flag_column, deleted_flag_column, is_active)
VALUES
('{Entity}{Group}Features',
 'Engineered {group} features for {entity} — normalised to [0,1] for ML training. No raw Domain values.',
 'Prediction', '{PredictionDatabase}',
 '{entity}_{group}_features', 'v_{entity}_features_current',
 'feature_group_key', NULL,
 'BI_TEMPORAL', 'is_current', NULL, 'Y');
```

**column_metadata** — register each feature with its computation logic:
```sql
INSERT INTO Semantic.column_metadata
(database_name, table_name, column_name,
 business_description, is_pii, is_sensitive, data_classification,
 is_required, data_type, is_active)
VALUES
('{PredictionDatabase}', '{entity}_{group}_features', '{feature_name}',
 'ENGINEERED: {plain English description}. Formula: {computation logic}. Scale: [0,1].',
 'N', 'N', 'INTERNAL', 'Y', 'DECIMAL(5,4)', 'Y');
-- Repeat for each engineered feature column
```

**table_relationship** — FK from Prediction to Domain:
```sql
INSERT INTO Semantic.table_relationship
(relationship_name, relationship_description,
 source_database, source_table, source_column,
 target_database, target_table, target_column,
 relationship_type, cardinality, relationship_meaning,
 is_mandatory, is_active)
VALUES
('{Entity}Features_To_{Entity}',
 'Prediction features reference Domain entity for context — join for raw attributes',
 '{PredictionDatabase}', '{entity}_{group}_features', 'entity_key',
 '{DomainDatabase}', '{Entity}_H', '{entity}_key',
 'FOREIGN_KEY', 'M:1',
 'Each feature snapshot belongs to one Domain entity; join for non-engineered attributes',
 'Y', 'Y');
```

**data_product_map** — update Prediction module status:
```sql
UPDATE Semantic.data_product_map
SET deployment_status = 'DEPLOYED',
    deployed_dts      = CURRENT_TIMESTAMP(6),
    primary_tables    = '{entity}_{group}_features, model_prediction',
    primary_views     = 'v_{entity}_features_current, v_{entity}_features_enriched',
    updated_at        = CURRENT_TIMESTAMP(6)
WHERE module_name = 'Prediction' AND is_active = 'Y';
```

---

## 7. Documentation Capture Checklist

- [ ] `dp_documentation.Module_Registry` INSERT generated (module_name = 'PREDICTION', version 1.0.0)
- [ ] `dp_documentation.Design_Decision` INSERTs generated (minimum 3 decisions)
- [ ] Decision IDs follow `DD-PREDICTION-{NNN}` convention
- [ ] Every decision has all ADR fields: context, alternatives_considered, rationale, consequences
- [ ] Decision categories from standard set: ARCHITECTURE | SCHEMA | NAMING | PERFORMANCE | SECURITY | INTEGRATION | OPERATIONAL
- [ ] All decisions: `decision_status = 'ACCEPTED'`, `is_current = 1`
- [ ] `dp_documentation.Change_Log` INSERT generated (CL-PREDICTION-001, INITIAL_RELEASE)
- [ ] Change log includes `migration_steps` and `rollback_steps`
- [ ] `dp_documentation.Business_Glossary` INSERTs generated (minimum 3 terms)
- [ ] `dp_documentation.Query_Cookbook` INSERTs generated (minimum 1 recipe)
- [ ] All INSERTs use consistent `data_product` value
- [ ] All temporal fields: `valid_from = CURRENT_DATE`, `valid_to = DATE '9999-12-31'`
- [ ] `source_module` and `module_version` populated on every row

---

## 8. Final Pre-Delivery Checklist

- [ ] Anti-pattern audit run — no unexplained raw Domain values in feature tables
- [ ] All features have computation logic documented in `COMMENT ON COLUMN`
- [ ] `is_current` is `BYTEINT NOT NULL DEFAULT 1` in every table (not `CHAR(1)`)
- [ ] Point-in-time correctness test passes (no future data leakage)
- [ ] Three standard views created and tested for each feature group
- [ ] Low-latency duplication exceptions documented per column (if any)
- [ ] Feature refresh strategy documented and process designed
- [ ] Semantic module updated: entity_metadata, column_metadata, table_relationship, data_product_map
- [ ] Training dataset generation query tested against real data
- [ ] Documentation capture complete — all dp_documentation INSERTs generated
