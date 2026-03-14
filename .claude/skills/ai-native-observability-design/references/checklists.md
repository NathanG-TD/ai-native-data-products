# Design Review Checklists
## AI-Native Observability Module

---

## 1. Designer Requirements Checklist

Before writing DDL, confirm:

- [ ] Tables/modules to track changes in identified
- [ ] Quality metrics and thresholds defined per table/column
- [ ] ML models to monitor identified (if Prediction module deployed)
- [ ] Agent outcome tracking needed? Action types and feedback categories defined
- [ ] OpenLineage integration in scope? Namespace and job naming convention agreed
- [ ] Retention policy defined per table (regulatory/operational requirements)
- [ ] Partitioning strategy decided based on event volume expectations
- [ ] Memory module feed scheduled? (frequency and minimum sample size agreed)

---

## 2. Structural Checklist

**All Observability tables:**
- [ ] `GENERATED ALWAYS AS IDENTITY` surrogate key
- [ ] **No `is_current` column** — Observability tables are append-only logs
- [ ] **No `is_deleted` column** — events are immutable once written
- [ ] All boolean flags: `BYTEINT NOT NULL DEFAULT value` with `is_` prefix
- [ ] `COMMENT ON TABLE` — explains what is monitored and the tracking level (table vs record)
- [ ] `COMMENT ON COLUMN` — all columns have meaningful metadata

**Boolean flag verification** (the corrected conventions):
- [ ] `data_quality_metric.is_threshold_met` — `BYTEINT NOT NULL DEFAULT 0`
- [ ] `model_performance.is_sla_met` — `BYTEINT NOT NULL DEFAULT 0`
- [ ] No `meets_threshold` or `meets_sla` column names (old `CHAR(1)` pattern)

**change_event:**
- [ ] `change_type` constrained to: INSERT | UPDATE | DELETE | MERGE | TRUNCATE
- [ ] `change_source` constrained to: ETL | API | MANUAL | AGENT
- [ ] `records_affected` — aggregate count, confirmed no individual record identifiers

**data_quality_metric:**
- [ ] `metric_name` uses standard names: COMPLETENESS, VALIDITY, UNIQUENESS, TIMELINESS, CONSISTENCY, ACCURACY
- [ ] `quality_threshold` populated for all metrics (not NULL)
- [ ] `is_threshold_met` computed correctly at INSERT time

**data_lineage:**
- [ ] `run_status` constrained to: SUCCESS | FAILED | PARTIAL
- [ ] `transformation_type` documented for all ETL/ELT patterns in use
- [ ] OpenLineage columns nullable (not required when external integration absent)

**model_performance:**
- [ ] `model_id` + `model_version` match values used in `Prediction.model_prediction`
- [ ] `is_sla_met` computed correctly at INSERT time

**agent_outcome:**
- [ ] `tables_accessed` stores table names only (no record identifiers)
- [ ] `session_id` can join to Memory.agent_session (if Memory module deployed)
- [ ] `user_feedback` constrained to: POSITIVE | NEUTRAL | NEGATIVE | CORRECTION

---

## 3. Anti-Pattern Audit — Business Data Check

```sql
-- Verify Observability tables contain NO business entity data
-- Check column names for suspicious patterns
SELECT TableName, ColumnName, ColumnType
FROM DBC.ColumnsV
WHERE DatabaseName = '{ObservabilityDatabase}'
  AND ColumnName NOT IN (
    -- Expected Observability column names
    'change_event_key', 'database_name', 'table_name', 'change_type',
    'change_dts', 'changed_by', 'change_reason', 'change_source',
    'records_affected', 'columns_changed', 'batch_id', 'job_name',
    'quality_metric_key', 'column_name', 'metric_name', 'metric_value',
    'metric_category', 'measured_dts', 'quality_threshold', 'is_threshold_met', 'sample_size',
    'lineage_key', 'source_database', 'source_table', 'source_system',
    'target_database', 'target_table', 'transformation_type',
    'transformation_logic', 'run_dts', 'openlineage_run_id',
    'openlineage_job_name', 'openlineage_namespace',
    'records_read', 'records_written', 'run_status',
    'performance_key', 'model_id', 'model_version', 'evaluation_dts', 'is_sla_met',
    'outcome_key', 'agent_id', 'session_id', 'action_type', 'action_dts',
    'tables_accessed', 'outcome_status', 'user_feedback',
    'execution_time_ms', 'records_processed',
    'created_at'
  )
ORDER BY TableName, ColumnName;
-- Any unexpected column may indicate business data leaking into Observability
```

---

## 4. Functional Validation Tests

```sql
-- Test 1: change_event accepts aggregate batch record (not per-record)
INSERT INTO Observability.change_event
(database_name, table_name, change_type, change_dts,
 changed_by, change_source, records_affected, batch_id, job_name)
VALUES
('TEST_DB', 'TEST_TABLE', 'INSERT', CURRENT_TIMESTAMP(6),
 'TEST_USER', 'MANUAL', 1000, 'TEST-BATCH-001', 'validation_test');
-- Expected: 1 row inserted

-- Test 2: data_quality_metric is_threshold_met logic correct
INSERT INTO Observability.data_quality_metric
(database_name, table_name, metric_name, metric_value,
 metric_category, measured_dts, quality_threshold, is_threshold_met, sample_size)
VALUES
('TEST_DB', 'TEST_TABLE', 'COMPLETENESS', 0.93,
 'COMPLETENESS', CURRENT_TIMESTAMP(6), 0.95, 0, 10000);
-- Expected: is_threshold_met = 0 (0.93 < 0.95 threshold)

-- Test 3: Quality failure detection query returns the test row
SELECT table_name, metric_name, metric_value, quality_threshold
FROM Observability.data_quality_metric
WHERE is_threshold_met = 0
  AND table_name = 'TEST_TABLE';
-- Expected: 1 row returned

-- Test 4: model_performance is_sla_met consistent with is_threshold_met pattern
INSERT INTO Observability.model_performance
(model_id, model_version, metric_name, metric_value,
 evaluation_dts, sample_size, is_sla_met)
VALUES
('test_model', 'v1', 'AUC', 0.72, CURRENT_TIMESTAMP(6), 5000, 0);
-- Expected: 1 row inserted (is_sla_met = 0 for a failing model)

-- Cleanup test records
DELETE FROM Observability.change_event WHERE batch_id = 'TEST-BATCH-001';
DELETE FROM Observability.data_quality_metric WHERE table_name = 'TEST_TABLE';
DELETE FROM Observability.model_performance WHERE model_id = 'test_model';
```

---

## 5. Retention Configuration Checklist

- [ ] Retention policy documented per table:
  - `change_event`: __ years (regulatory audit requirement)
  - `data_quality_metric`: __ months/years (trend analysis requirement)
  - `data_lineage`: __ years (lineage traceability requirement)
  - `model_performance`: lifetime of model + __ months
  - `agent_outcome`: __ months rolling window
- [ ] Partitioning applied to tables with high event volume
- [ ] Archive/purge process designed (Teradata FastExport + DELETE or partition drop)
- [ ] Partition boundaries cover expected retention period + growth buffer

---

## 6. Semantic Module Registration Checklist

```sql
-- entity_metadata: one row per Observability table
INSERT INTO Semantic.entity_metadata
(entity_name, entity_description, module_name, database_name,
 table_name, view_name, surrogate_key_column, natural_key_column,
 temporal_pattern, current_flag_column, deleted_flag_column, is_active)
VALUES
('ChangeEvent',
 'Table-level audit trail — one row per batch operation tracking what changed, when, and by whom',
 'Observability', '{ObservabilityDatabase}',
 'change_event', NULL, 'change_event_key', NULL,
 'NONE', NULL, NULL, 'Y'),

('DataQualityMetric',
 'Quality scores by table and column — tracks completeness, validity, timeliness over time',
 'Observability', '{ObservabilityDatabase}',
 'data_quality_metric', NULL, 'quality_metric_key', NULL,
 'NONE', NULL, NULL, 'Y'),

('DataLineage',
 'Data provenance records — source-to-target transformation history, OpenLineage aligned',
 'Observability', '{ObservabilityDatabase}',
 'data_lineage', NULL, 'lineage_key', NULL,
 'NONE', NULL, NULL, 'Y'),

('ModelPerformance',
 'ML model performance metrics over time — AUC, drift, latency, SLA tracking',
 'Observability', '{ObservabilityDatabase}',
 'model_performance', NULL, 'performance_key', NULL,
 'NONE', NULL, NULL, 'Y'),

('AgentOutcome',
 'Agent action outcomes and user feedback — drives closed-loop learning into Memory module',
 'Observability', '{ObservabilityDatabase}',
 'agent_outcome', NULL, 'outcome_key', NULL,
 'NONE', NULL, NULL, 'Y');

-- data_product_map: update Observability status
UPDATE Semantic.data_product_map
SET deployment_status = 'DEPLOYED',
    deployed_dts      = CURRENT_TIMESTAMP(6),
    primary_tables    = 'change_event, data_quality_metric, data_lineage, model_performance, agent_outcome',
    primary_views     = NULL,
    updated_at        = CURRENT_TIMESTAMP(6)
WHERE module_name = 'Observability' AND is_active = 'Y';
```

---

## 7. Documentation Capture Checklist

- [ ] Documentation SQL written inline as `05-observability/{NN}-documentation.sql` (last numbered file in module directory)
- [ ] File header includes `-- Deploy: Phase 3, after Observability DDL` comment
- [ ] `dp_documentation.Module_Registry` INSERT generated (module_name = 'OBSERVABILITY', version 1.0.0)
- [ ] `dp_documentation.Design_Decision` INSERTs generated (minimum 3 decisions)
- [ ] Decision IDs follow `DD-OBSERVABILITY-{NNN}` convention
- [ ] Every decision has all ADR fields: context, alternatives_considered, rationale, consequences
- [ ] Decision categories from standard set: ARCHITECTURE | SCHEMA | NAMING | PERFORMANCE | SECURITY | INTEGRATION | OPERATIONAL
- [ ] All decisions: `decision_status = 'ACCEPTED'`, `is_current = 1`
- [ ] `dp_documentation.Change_Log` INSERT generated (CL-OBSERVABILITY-001, INITIAL_RELEASE)
- [ ] Change log includes `migration_steps` and `rollback_steps`
- [ ] `dp_documentation.Business_Glossary` INSERTs generated (minimum 3 terms)
- [ ] `dp_documentation.Query_Cookbook` INSERTs generated (minimum 1 recipe)
- [ ] All INSERTs use consistent `data_product` value
- [ ] All temporal fields: `valid_from = CURRENT_DATE`, `valid_to = DATE '9999-12-31'`
- [ ] `source_module` and `module_version` populated on every row

---

## 8. Final Pre-Delivery Checklist

- [ ] All five tables created with correct boolean conventions (`BYTEINT`, `is_` prefix)
- [ ] No `is_current` or `is_deleted` columns in any Observability table
- [ ] Anti-pattern audit run — no business entity data in Observability tables
- [ ] Functional validation tests pass (insert + query for each table)
- [ ] Retention policy documented and partitioning applied where needed
- [ ] Memory feed query designed and scheduled (if Memory module deployed)
- [ ] OpenLineage column population confirmed with pipeline team (if in scope)
- [ ] Semantic module updated: entity_metadata × 5 tables, data_product_map updated
- [ ] Quality thresholds agreed with data owners and set in `quality_threshold` column
- [ ] Documentation capture complete — all dp_documentation INSERTs generated
