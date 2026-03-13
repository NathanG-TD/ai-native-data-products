# Observability Query Patterns Reference
## Monitoring, Memory Feed, Cross-Module Health, Agent Discovery

---

## 1. Change Tracking Queries

### Table Change History

```sql
-- Recent changes to a specific table
SELECT
    change_type,
    change_dts,
    changed_by,
    change_source,
    records_affected,
    batch_id,
    job_name
FROM Observability.change_event
WHERE table_name = '{TableName}'
  AND change_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '30' DAY
ORDER BY change_dts DESC;

-- Change volume trend by table (last 7 days)
SELECT
    table_name,
    CAST(change_dts AS DATE) AS change_date,
    change_type,
    SUM(records_affected) AS total_records,
    COUNT(*) AS batch_count
FROM Observability.change_event
WHERE change_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '7' DAY
GROUP BY table_name, CAST(change_dts AS DATE), change_type
ORDER BY table_name, change_date DESC;

-- Changes by source (ETL vs API vs AGENT vs MANUAL)
SELECT
    change_source,
    COUNT(*) AS event_cnt,
    SUM(records_affected) AS total_records_affected
FROM Observability.change_event
WHERE change_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '30' DAY
GROUP BY change_source
ORDER BY total_records_affected DESC;
```

---

## 2. Data Quality Monitoring

### Current Quality Status Across All Tables

```sql
-- Latest quality score per table + column
SELECT
    database_name,
    table_name,
    column_name,
    metric_name,
    metric_value,
    quality_threshold,
    is_threshold_met,
    measured_dts
FROM Observability.data_quality_metric dqm
WHERE measured_dts = (
    SELECT MAX(measured_dts)
    FROM Observability.data_quality_metric
    WHERE table_name  = dqm.table_name
      AND metric_name = dqm.metric_name
      AND COALESCE(column_name, '') = COALESCE(dqm.column_name, '')
)
ORDER BY is_threshold_met ASC, metric_value ASC;  -- Failures first
```

### Quality Failures (Alerting)

```sql
-- All metrics currently below threshold — use for alerting
SELECT
    database_name,
    table_name,
    column_name,
    metric_name,
    metric_value,
    quality_threshold,
    metric_value - quality_threshold AS shortfall,
    measured_dts
FROM Observability.data_quality_metric
WHERE is_threshold_met = 0
  AND measured_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '24' HOUR
ORDER BY shortfall ASC;  -- Worst failures first
```

### Quality Trend (Is It Getting Better or Worse?)

```sql
-- Weekly quality trend for a specific metric
SELECT
    table_name,
    metric_name,
    CAST(measured_dts AS DATE) AS measure_date,
    AVG(metric_value) AS avg_score,
    MIN(metric_value) AS min_score,
    SUM(CASE WHEN is_threshold_met = 0 THEN 1 ELSE 0 END) AS failure_cnt
FROM Observability.data_quality_metric
WHERE table_name  = '{TableName}'
  AND metric_name = '{MetricName}'
  AND measured_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '90' DAY
GROUP BY table_name, metric_name, CAST(measured_dts AS DATE)
ORDER BY measure_date DESC;
```

---

## 3. Data Lineage Queries

```sql
-- Full lineage chain for a target table (what feeds it?)
SELECT
    source_database || '.' || source_table AS source,
    target_database || '.' || target_table AS target,
    transformation_type,
    job_name,
    records_read,
    records_written,
    run_status,
    run_dts
FROM Observability.data_lineage
WHERE target_table = '{TargetTable}'
  AND run_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '7' DAY
ORDER BY run_dts DESC;

-- Failed transformation runs (need investigation)
SELECT
    source_database || '.' || source_table AS source,
    target_database || '.' || target_table AS target,
    job_name,
    run_dts,
    records_read,
    records_written
FROM Observability.data_lineage
WHERE run_status = 'FAILED'
  AND run_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '24' HOUR
ORDER BY run_dts DESC;

-- OpenLineage correlation (for external catalog cross-reference)
SELECT
    openlineage_run_id,
    openlineage_job_name,
    openlineage_namespace,
    source_table,
    target_table,
    run_status,
    run_dts
FROM Observability.data_lineage
WHERE openlineage_run_id IS NOT NULL
  AND run_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '1' DAY
ORDER BY run_dts DESC;
```

---

## 4. Model Performance Monitoring

```sql
-- Current performance for all models (latest measurement per model + metric)
SELECT
    model_id,
    model_version,
    metric_name,
    metric_value,
    is_sla_met,
    sample_size,
    evaluation_dts
FROM Observability.model_performance mp
WHERE evaluation_dts = (
    SELECT MAX(evaluation_dts)
    FROM Observability.model_performance
    WHERE model_id    = mp.model_id
      AND metric_name = mp.metric_name
)
ORDER BY is_sla_met ASC, model_id, metric_name;  -- Failures first

-- Model performance trend (detecting drift)
SELECT
    model_id,
    model_version,
    metric_name,
    CAST(evaluation_dts AS DATE) AS eval_date,
    AVG(metric_value) AS avg_metric,
    SUM(CASE WHEN is_sla_met = 0 THEN 1 ELSE 0 END) AS sla_failures
FROM Observability.model_performance
WHERE model_id    = '{ModelId}'
  AND metric_name IN ('AUC', 'DRIFT_SCORE', 'ACCURACY')
  AND evaluation_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '90' DAY
GROUP BY model_id, model_version, metric_name, CAST(evaluation_dts AS DATE)
ORDER BY eval_date DESC;
```

---

## 5. Agent Outcome Analysis

```sql
-- Agent success rate by action type (last 30 days)
SELECT
    action_type,
    COUNT(*) AS total_actions,
    SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS success_cnt,
    SUM(CASE WHEN outcome_status = 'FAILED'  THEN 1 ELSE 0 END) AS failure_cnt,
    CAST(SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS DECIMAL(10,4))
        / COUNT(*) AS success_rate,
    AVG(execution_time_ms) AS avg_latency_ms
FROM Observability.agent_outcome
WHERE action_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '30' DAY
GROUP BY action_type
ORDER BY success_rate ASC;  -- Worst performing first

-- User feedback distribution
SELECT
    user_feedback,
    COUNT(*) AS feedback_cnt,
    CAST(COUNT(*) AS DECIMAL(10,4)) / SUM(COUNT(*)) OVER () AS feedback_pct
FROM Observability.agent_outcome
WHERE user_feedback IS NOT NULL
  AND action_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '30' DAY
GROUP BY user_feedback
ORDER BY feedback_cnt DESC;

-- Most accessed tables by agents (optimisation signals)
SELECT
    TRIM(table_ref) AS table_accessed,
    COUNT(*) AS access_cnt,
    AVG(execution_time_ms) AS avg_latency_ms
FROM Observability.agent_outcome
    -- Expand comma-separated tables_accessed — one row per table per outcome
    CROSS JOIN TABLE (
        STRTOK_SPLIT_TO_TABLE(tables_accessed, ',')
        RETURNS (token_num INTEGER, table_ref VARCHAR(200))
    ) AS tbl
WHERE action_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '30' DAY
GROUP BY TRIM(table_ref)
ORDER BY access_cnt DESC;
```

---

## 6. Memory Feed Pattern (Closed-Loop Learning)

Observability drives continuous improvement by identifying successful patterns and writing them to Memory.

```sql
-- Identify high-performing query patterns and persist to Memory
-- Run this as a scheduled job (daily or weekly)

INSERT INTO Memory.learned_strategy (
    strategy_name,
    strategy_category,
    strategy_description,
    success_rate,
    sample_size,
    discovered_by_agent,
    scope_level,
    first_observed_dts,
    last_observed_dts
)
SELECT
    'HighPerformance_' || action_type || '_' || CAST(CURRENT_DATE AS VARCHAR(10)) AS strategy_name,
    'QUERY_OPTIMIZATION' AS strategy_category,
    'Action type ' || action_type || ' on tables: ' ||
        MIN(tables_accessed) || ' averaged ' ||
        CAST(AVG(execution_time_ms) AS VARCHAR(10)) || 'ms with high success' AS strategy_description,
    CAST(SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS DECIMAL(10,4))
        / COUNT(*) AS success_rate,
    COUNT(*) AS sample_size,
    'observability_analyzer' AS discovered_by_agent,
    'ORGANIZATION' AS scope_level,
    MIN(action_dts) AS first_observed_dts,
    MAX(action_dts) AS last_observed_dts
FROM Observability.agent_outcome
WHERE action_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '7' DAY
  AND outcome_status = 'SUCCESS'
  AND execution_time_ms < 1000  -- Fast executions
GROUP BY action_type
HAVING COUNT(*) >= 10  -- Minimum sample for statistical reliability
   AND CAST(SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS DECIMAL(10,4))
       / COUNT(*) >= 0.9;  -- 90%+ success rate
```

---

## 7. Cross-Module Health Dashboard

Single query summarising data product health across all modules:

```sql
-- Overall data product health summary
SELECT 'DATA_QUALITY' AS metric_category,
    table_name,
    metric_name,
    CAST(AVG(metric_value) AS DECIMAL(5,4)) AS avg_score,
    SUM(CASE WHEN is_threshold_met = 0 THEN 1 ELSE 0 END) AS failures_7d
FROM Observability.data_quality_metric
WHERE measured_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '7' DAY
GROUP BY table_name, metric_name

UNION ALL

SELECT 'MODEL_PERFORMANCE',
    model_id,
    metric_name,
    CAST(AVG(metric_value) AS DECIMAL(5,4)),
    SUM(CASE WHEN is_sla_met = 0 THEN 1 ELSE 0 END)
FROM Observability.model_performance
WHERE evaluation_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '7' DAY
GROUP BY model_id, metric_name

UNION ALL

SELECT 'AGENT_OUTCOMES',
    action_type,
    'SUCCESS_RATE',
    CAST(SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS DECIMAL(10,4))
        / COUNT(*),
    SUM(CASE WHEN outcome_status = 'FAILED' THEN 1 ELSE 0 END)
FROM Observability.agent_outcome
WHERE action_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '7' DAY
GROUP BY action_type

ORDER BY failures_7d DESC, avg_score ASC;
```

---

## 8. Recording Observability Events

### change_event Population Pattern

```sql
-- Record a batch ETL load (ALWAYS use aggregate, not per-record)
INSERT INTO Observability.change_event
(database_name, table_name, change_type, change_dts,
 changed_by, change_source, records_affected,
 columns_changed, batch_id, job_name)
VALUES
('{TargetDatabase}', '{TargetTable}', 'INSERT', CURRENT_TIMESTAMP(6),
 '{ETLJobName}', 'ETL', {record_count},
 '{comma_separated_column_names}', '{batch_id}', '{job_name}');
```

### data_quality_metric Population Pattern

```sql
-- Record completeness measurement for a column
INSERT INTO Observability.data_quality_metric
(database_name, table_name, column_name,
 metric_name, metric_value, metric_category,
 measured_dts, quality_threshold, is_threshold_met, sample_size)
SELECT
    '{Database}', '{Table}', '{Column}',
    'COMPLETENESS',
    CAST(COUNT({Column}) AS DECIMAL(10,4)) / COUNT(*),  -- non-null / total
    'COMPLETENESS',
    CURRENT_TIMESTAMP(6),
    0.95,  -- threshold: 95% completeness required
    CASE WHEN CAST(COUNT({Column}) AS DECIMAL(10,4)) / COUNT(*) >= 0.95
         THEN 1 ELSE 0 END,
    COUNT(*)
FROM {Database}.{Table}
WHERE is_current = 1;  -- measure against current active records
```
