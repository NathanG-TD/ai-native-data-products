# Observability Tables Reference
## DDL Templates for AI-Native Observability Module

Replace `Observability` with your actual database name (e.g., `Customer360_Observability`).

**Key conventions**:
- All tables are **append-only** — no `is_current`, no soft delete, no temporal versioning
- All boolean flags: `BYTEINT NOT NULL DEFAULT value` with `is_` prefix
- Events reference tables by name (strings), not by FK to domain keys — keeps Observability decoupled

---

## 1. change_event — Audit Trail

Table-level change tracking. One row per batch operation, not one per record changed.

```sql
CREATE TABLE Observability.change_event (
    change_event_key    INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for change event record',

    -- What changed (table level — not individual records)
    database_name       VARCHAR(100)
        COMMENT 'Database where change occurred',
    table_name          VARCHAR(100) NOT NULL
        COMMENT 'Table that was changed — table-level tracking, not individual record tracking',

    -- Change details
    change_type         VARCHAR(20) NOT NULL
        COMMENT 'Change operation: INSERT | UPDATE | DELETE | MERGE | TRUNCATE',
    change_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When the change occurred',

    -- Who and why
    changed_by          VARCHAR(100) NOT NULL
        COMMENT 'User or process that made the change: ETL_NIGHTLY, john.doe, API_SERVICE, AGENT',
    change_reason       VARCHAR(500)
        COMMENT 'Business justification or trigger for this change',
    change_source       VARCHAR(50)
        COMMENT 'Source category: ETL | API | MANUAL | AGENT',

    -- Aggregate metrics (never individual record identifiers)
    records_affected    INTEGER
        COMMENT 'Number of records affected — aggregate count for the batch, not individual keys',
    columns_changed     VARCHAR(1000)
        COMMENT 'Comma-separated list of column names modified in this change',

    -- Batch / job tracking
    batch_id            VARCHAR(100)
        COMMENT 'Batch identifier — links related changes in the same ETL run or transaction',
    job_name            VARCHAR(200)
        COMMENT 'ETL job, API endpoint, or process name that made the change',

    -- Audit
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this change event record was created in Observability'
)
PRIMARY INDEX (change_event_key);

COMMENT ON TABLE Observability.change_event IS
'Table-level audit trail — one row per batch operation, never one per individual record.
 Tracks what changed, when, by whom, and how many records were affected.
 Query by table_name + change_dts range for operational audit and lineage analysis.';
```

**Correct usage example:**
```sql
-- ✅ ONE event for a batch of 125,000 record updates
INSERT INTO Observability.change_event
(database_name, table_name, change_type, change_dts,
 changed_by, change_source, records_affected, batch_id, job_name)
VALUES
('Customer360_Domain', 'Party_H', 'UPDATE', CURRENT_TIMESTAMP(6),
 'ETL_NIGHTLY', 'ETL', 125000, 'BATCH-2024-03-15-001', 'nightly_party_load');

-- ❌ NOT 125,000 separate INSERT statements — that defeats the purpose
```

---

## 2. data_quality_metric — Quality Monitoring

```sql
CREATE TABLE Observability.data_quality_metric (
    quality_metric_key  INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for quality metric record',

    -- What was measured
    database_name       VARCHAR(100)
        COMMENT 'Database containing the measured table',
    table_name          VARCHAR(100) NOT NULL
        COMMENT 'Table being quality-measured',
    column_name         VARCHAR(100)
        COMMENT 'Column being measured — NULL for table-level metrics',

    -- The metric
    metric_name         VARCHAR(100) NOT NULL
        COMMENT 'Quality dimension: COMPLETENESS | VALIDITY | UNIQUENESS | TIMELINESS | CONSISTENCY | ACCURACY',
    metric_value        DECIMAL(10,4)
        COMMENT 'Measured value — typically 0.0 to 1.0 (percentage as decimal)',
    metric_category     VARCHAR(50)
        COMMENT 'Metric grouping: COMPLETENESS | VALIDITY | CONSISTENCY | TIMELINESS | ACCURACY',

    -- Assessment
    measured_dts        TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When metric was measured — enables trend analysis over time',
    quality_threshold   DECIMAL(5,4)
        COMMENT 'Minimum acceptable value for this metric — e.g. 0.95 for 95% completeness',
    is_threshold_met    BYTEINT NOT NULL DEFAULT 0
        COMMENT 'Threshold result: 1 = metric_value >= quality_threshold (pass), 0 = below threshold (fail)',
    sample_size         INTEGER
        COMMENT 'Number of records analysed for this measurement',

    -- Audit
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this quality metric record was created'
)
PRIMARY INDEX (quality_metric_key);

COMMENT ON TABLE Observability.data_quality_metric IS
'Data quality metrics by table and column — tracks quality scores over time for trend monitoring and alerting.
 is_threshold_met = 1 means the metric passed its defined threshold; 0 means it failed.
 Standard metric_name values: COMPLETENESS, VALIDITY, UNIQUENESS, TIMELINESS, CONSISTENCY, ACCURACY.
 Compatible with Great Expectations, Deequ, and Monte Carlo naming conventions.';
```

---

## 3. data_lineage — Data Provenance (OpenLineage Aligned)

```sql
CREATE TABLE Observability.data_lineage (
    lineage_key             INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for lineage record',

    -- Source
    source_database         VARCHAR(100)
        COMMENT 'Source database — where input data came from (NULL if external system)',
    source_table            VARCHAR(100)
        COMMENT 'Source table — input table for this transformation',
    source_system           VARCHAR(100)
        COMMENT 'External source system name — originating system if outside Teradata',

    -- Target
    target_database         VARCHAR(100)
        COMMENT 'Target database — where output data was written',
    target_table            VARCHAR(100) NOT NULL
        COMMENT 'Target table — output table from transformation',

    -- Transformation
    transformation_type     VARCHAR(50)
        COMMENT 'Transformation category: ETL | FEATURE_ENG | AGGREGATION | JOIN | EMBEDDING_GEN | PREDICTION',
    transformation_logic    VARCHAR(4000)
        COMMENT 'SQL, algorithm, or process description for this transformation',
    job_name                VARCHAR(200)
        COMMENT 'Job or process name that performed the transformation',
    run_dts                 TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When the transformation executed',

    -- OpenLineage correlation (for external catalog integration)
    openlineage_run_id      VARCHAR(200)
        COMMENT 'OpenLineage run UUID — correlates with Marquez, Amundsen, or other lineage catalogs',
    openlineage_job_name    VARCHAR(200)
        COMMENT 'OpenLineage job name in standard format',
    openlineage_namespace   VARCHAR(200)
        COMMENT 'OpenLineage namespace — environment identifier: production, staging, dev',

    -- Volume metrics
    records_read            INTEGER
        COMMENT 'Number of records read from source — input volume metric',
    records_written         INTEGER
        COMMENT 'Number of records written to target — output volume metric',
    run_status              VARCHAR(20)
        COMMENT 'Execution outcome: SUCCESS | FAILED | PARTIAL',

    -- Audit
    created_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this lineage record was created'
)
PRIMARY INDEX (lineage_key);

COMMENT ON TABLE Observability.data_lineage IS
'Data lineage and provenance — records source-to-target transformations aligned with OpenLineage standard.
 Store openlineage_run_id when integrating with external lineage catalogs (Marquez, Amundsen).
 transformation_type categories: ETL, FEATURE_ENG, AGGREGATION, JOIN, EMBEDDING_GEN, PREDICTION.
 run_status: SUCCESS | FAILED | PARTIAL.';
```

---

## 4. model_performance — ML Model Monitoring

```sql
CREATE TABLE Observability.model_performance (
    performance_key     INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for performance metric record',

    -- Model identification
    model_id            VARCHAR(100) NOT NULL
        COMMENT 'Unique model identifier — matches model_id in Prediction.model_prediction',
    model_version       VARCHAR(20) NOT NULL
        COMMENT 'Model version — matches model_version in Prediction.model_prediction',

    -- The metric
    metric_name         VARCHAR(100) NOT NULL
        COMMENT 'Performance metric: ACCURACY | PRECISION | RECALL | AUC | F1 | LATENCY_MS | DRIFT_SCORE | FEATURE_DRIFT',
    metric_value        DECIMAL(10,6)
        COMMENT 'Measured performance value — interpretation depends on metric_name',

    -- Assessment
    evaluation_dts      TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When performance was measured',
    sample_size         INTEGER
        COMMENT 'Number of predictions evaluated for this metric',
    is_sla_met          BYTEINT NOT NULL DEFAULT 0
        COMMENT 'SLA result: 1 = metric meets performance SLA threshold (pass), 0 = below threshold (fail)',

    -- Audit
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this performance record was created'
)
PRIMARY INDEX (performance_key);

COMMENT ON TABLE Observability.model_performance IS
'ML model performance metrics over time — tracks accuracy, latency, and drift for monitoring and alerting.
 is_sla_met = 1 means the metric passed its SLA; 0 means it failed.
 Links to Prediction module via model_id + model_version.
 Metric categories: classification metrics (ACCURACY, AUC, F1), regression metrics, drift (DRIFT_SCORE, FEATURE_DRIFT), operational (LATENCY_MS).';
```

---

## 5. agent_outcome — Agent Feedback Loop

```sql
CREATE TABLE Observability.agent_outcome (
    outcome_key         INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for outcome record',

    -- Agent identification
    agent_id            VARCHAR(100) NOT NULL
        COMMENT 'Agent identifier — which agent performed this action',
    session_id          VARCHAR(100)
        COMMENT 'Session identifier — links outcome to agent session in Memory module',

    -- Action
    action_type         VARCHAR(50) NOT NULL
        COMMENT 'Action category: QUERY | RECOMMENDATION | DECISION | PREDICTION | ANALYSIS',
    action_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When the agent action was performed',
    tables_accessed     VARCHAR(1000)
        COMMENT 'Comma-separated database.table names involved — table-level tracking only, no individual record keys',

    -- Outcome
    outcome_status      VARCHAR(20) NOT NULL
        COMMENT 'Action result: SUCCESS | PARTIAL | FAILED',
    user_feedback       VARCHAR(20)
        COMMENT 'Human feedback received: POSITIVE | NEUTRAL | NEGATIVE | CORRECTION — enables closed-loop learning',
    execution_time_ms   INTEGER
        COMMENT 'Wall-clock execution time in milliseconds — latency metric for action',
    records_processed   INTEGER
        COMMENT 'Aggregate count of records involved — scale metric, never individual record identifiers',

    -- Audit
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this outcome record was created'
)
PRIMARY INDEX (outcome_key);

COMMENT ON TABLE Observability.agent_outcome IS
'Agent action outcomes and user feedback — enables closed-loop learning by tracking what worked.
 table-level tracking only: tables_accessed stores names, not individual record keys.
 user_feedback drives Memory module learning: POSITIVE/NEGATIVE outcomes become learned strategies.
 session_id links to Memory.agent_session for full session context.';
```

---

## 6. Partitioning Templates (for High-Volume Tables)

Apply when event volume warrants time-based partition elimination:

```sql
-- Partition change_event by month (common for ETL-heavy products)
CREATE TABLE Observability.change_event (
    -- [all columns as above]
)
PRIMARY INDEX (change_event_key)
PARTITION BY RANGE_N(
    change_dts BETWEEN TIMESTAMP '2024-01-01 00:00:00+00:00'
               AND     TIMESTAMP '2030-12-31 23:59:59+00:00'
               EACH INTERVAL '1' MONTH
);

-- Partition data_quality_metric by month
-- Partition agent_outcome by week (if high-frequency agent activity)
-- Apply same pattern — adjust interval to match event frequency
```
