# Observability Module Design Standard
## AI-Native Data Product Architecture - Version 1.0

---

## Document Control

| Attribute | Value |
|-----------|-------|
| **Version** | 1.1 |
| **Status** | STANDARD |
| **Last Updated** | 2025-02-27 |
| **Owner** | Nathan Green, Worldwide Data Architecture Team, Teradata |
| **Scope** | Observability Module (Monitoring & Feedback) |
| **Type** | Design Standard (Structural Requirements) |

---

## Table of Contents

1. [AI-Native Observability Module Overview](#1-ai-native-observability-module-overview)
2. [Module Scope and Boundaries](#2-module-scope-and-boundaries)
3. [Core Observability Tables](#3-core-observability-tables)
4. [Open Standards Integration](#4-open-standards-integration)
5. [Integration with Other Modules](#5-integration-with-other-modules)
6. [Designer Responsibilities](#6-designer-responsibilities)

---

## 1. AI-Native Observability Module Overview

### 1.1 Primary Purpose

The Observability Module monitors data product health and enables continuous improvement through outcome tracking and feedback loops.

**Key capabilities**:
- Data quality monitoring
- Change tracking (audit trail)
- Data lineage (provenance)
- Performance monitoring
- Outcome tracking

### 1.2 Critical Principle: Events and Metrics, Not Data

Observability stores **events and metrics**, not business data:
- ✅ "Party record 1001 was updated by john.doe at 2024-03-15"
- ❌ NOT the customer's actual details
- ✅ "Data quality score for Party_H: 0.95"
- ❌ NOT the actual Party records

---

## 2. Module Scope and Boundaries

**IN SCOPE**:
- Change events (what, when, who, why)
- Data quality metrics
- Data lineage (OpenLineage aligned)
- Performance metrics
- Outcome tracking

**OUT OF SCOPE**:
- ❌ Business domain data → Domain Module
- ❌ Query result sets → Not stored

---

## 3. Core Observability Tables

### 3.1 change_event (Table-Level Change Tracking)

```sql
CREATE TABLE Observability.change_event (
    change_event_key INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    
    -- What changed (TABLE LEVEL - not individual records)
    database_name VARCHAR(100),
    table_name VARCHAR(100) NOT NULL,
    
    -- Change details
    change_type VARCHAR(20) NOT NULL,  -- 'INSERT', 'UPDATE', 'DELETE', 'MERGE', 'TRUNCATE'
    change_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    
    -- Who and why
    changed_by VARCHAR(100) NOT NULL,
    change_reason VARCHAR(500),
    change_source VARCHAR(50),  -- 'ETL', 'API', 'MANUAL', 'AGENT'
    
    -- Aggregate metrics (table-level summary)
    records_affected INTEGER,  -- How many records changed
    columns_changed VARCHAR(1000),  -- Comma-separated list of columns modified
    
    -- Batch/job tracking
    batch_id VARCHAR(100),
    job_name VARCHAR(200),
    
    -- Metadata
    created_at TIMESTAMP(6) WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP(6)
)
PRIMARY INDEX (change_event_key);

COMMENT ON TABLE Observability.change_event IS 
'Table-level change event tracking - records aggregated changes for audit trail, NOT individual record details';

COMMENT ON COLUMN Observability.change_event.change_event_key IS 
'Surrogate key for change event record';

COMMENT ON COLUMN Observability.change_event.database_name IS 
'Database where change occurred';

COMMENT ON COLUMN Observability.change_event.table_name IS 
'Table that was changed - table-level tracking, not individual record tracking';

COMMENT ON COLUMN Observability.change_event.change_type IS 
'Type of change - INSERT (new records), UPDATE (modified), DELETE (removed), MERGE (upsert), TRUNCATE (full table clear)';

COMMENT ON COLUMN Observability.change_event.change_dts IS 
'Timestamp when change occurred';

COMMENT ON COLUMN Observability.change_event.changed_by IS 
'User or process that made the change - ETL_NIGHTLY, john.doe, API_SERVICE, etc.';

COMMENT ON COLUMN Observability.change_event.change_reason IS 
'Reason for change - business justification or trigger for modification';

COMMENT ON COLUMN Observability.change_event.change_source IS 
'Source of change - ETL (batch process), API (application), MANUAL (human user), AGENT (AI agent)';

COMMENT ON COLUMN Observability.change_event.records_affected IS 
'Number of records affected by change - aggregate count for table-level tracking, enables scale monitoring';

COMMENT ON COLUMN Observability.change_event.columns_changed IS 
'Columns modified - comma-separated list of column names that were updated';

COMMENT ON COLUMN Observability.change_event.batch_id IS 
'Batch identifier - links related changes in same ETL run or transaction';

COMMENT ON COLUMN Observability.change_event.job_name IS 
'Job or process name - identifies the ETL job, API endpoint, or process that made change';

COMMENT ON COLUMN Observability.change_event.created_at IS 
'Timestamp when change event record was created in Observability module';
```

**Example**:
```sql
-- ETL updates 125,000 Party records
INSERT INTO Observability.change_event (
    database_name, table_name, change_type, change_dts,
    changed_by, change_source, records_affected, batch_id
) VALUES (
    'Domain', 'Party_H', 'UPDATE', CURRENT_TIMESTAMP(6),
    'ETL_NIGHTLY', 'ETL', 125000, 'BATCH-2024-03-15-001'
);

-- NOT 125,000 separate change events!
```

**Benefits**:
- ✅ Consistent with Memory (table-level references)
- ✅ Scales efficiently (one event per batch, not per record)
- ✅ "Small answers" - aggregate metrics, not individual record tracking
- ✅ Sufficient for monitoring and audit (know what happened at table level)

### 3.2 data_quality_metric

```sql
CREATE TABLE Observability.data_quality_metric (
    quality_metric_key INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    database_name VARCHAR(100),
    table_name VARCHAR(100) NOT NULL,
    column_name VARCHAR(100),
    metric_name VARCHAR(100) NOT NULL,
    metric_value DECIMAL(10,4),
    metric_category VARCHAR(50),
    measured_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    quality_threshold DECIMAL(5,4),
    is_threshold_met BYTEINT NOT NULL DEFAULT 0,
    sample_size INTEGER,
    created_at TIMESTAMP(6) WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP(6)
)
PRIMARY INDEX (quality_metric_key);

COMMENT ON TABLE Observability.data_quality_metric IS 
'Data quality metrics by table and column - tracks quality trends over time for monitoring and alerting';

COMMENT ON COLUMN Observability.data_quality_metric.quality_metric_key IS 
'Surrogate key for quality metric record';

COMMENT ON COLUMN Observability.data_quality_metric.database_name IS 
'Database containing the measured table';

COMMENT ON COLUMN Observability.data_quality_metric.table_name IS 
'Table being measured for quality';

COMMENT ON COLUMN Observability.data_quality_metric.column_name IS 
'Column being measured - NULL for table-level metrics';

COMMENT ON COLUMN Observability.data_quality_metric.metric_name IS 
'Quality metric name - COMPLETENESS, VALIDITY, UNIQUENESS, TIMELINESS, CONSISTENCY, ACCURACY';

COMMENT ON COLUMN Observability.data_quality_metric.metric_value IS 
'Measured metric value - typically 0.0 to 1.0 range representing percentage or score';

COMMENT ON COLUMN Observability.data_quality_metric.metric_category IS 
'Metric category - COMPLETENESS, VALIDITY, CONSISTENCY, TIMELINESS - groups related metrics';

COMMENT ON COLUMN Observability.data_quality_metric.measured_dts IS 
'When metric was measured - enables quality trend analysis over time';

COMMENT ON COLUMN Observability.data_quality_metric.quality_threshold IS 
'Expected quality threshold - minimum acceptable value for this metric';

COMMENT ON COLUMN Observability.data_quality_metric.is_threshold_met IS 
'Threshold met indicator - 1 = passes quality check, 0 = fails quality check - enables alerting';

COMMENT ON COLUMN Observability.data_quality_metric.sample_size IS 
'Sample size used for measurement - number of records analyzed';

COMMENT ON COLUMN Observability.data_quality_metric.created_at IS 
'Timestamp when quality metric record was created';
```

### 3.3 data_lineage (OpenLineage Aligned)

```sql
CREATE TABLE Observability.data_lineage (
    lineage_key INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    source_database VARCHAR(100),
    source_table VARCHAR(100),
    source_system VARCHAR(100),
    target_database VARCHAR(100),
    target_table VARCHAR(100) NOT NULL,
    transformation_type VARCHAR(50),
    transformation_logic VARCHAR(4000),
    job_name VARCHAR(200),
    run_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    openlineage_run_id VARCHAR(200),
    openlineage_job_name VARCHAR(200),
    openlineage_namespace VARCHAR(200),
    records_read INTEGER,
    records_written INTEGER,
    run_status VARCHAR(20),
    created_at TIMESTAMP(6) WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP(6)
)
PRIMARY INDEX (lineage_key);

COMMENT ON TABLE Observability.data_lineage IS 
'Data lineage tracking - records data provenance and transformations aligned with OpenLineage standard';

COMMENT ON COLUMN Observability.data_lineage.lineage_key IS 
'Surrogate key for lineage record';

COMMENT ON COLUMN Observability.data_lineage.source_database IS 
'Source database name - where input data came from';

COMMENT ON COLUMN Observability.data_lineage.source_table IS 
'Source table name - input table for transformation';

COMMENT ON COLUMN Observability.data_lineage.source_system IS 
'External source system name - originating system for data, NULL if internal Teradata source';

COMMENT ON COLUMN Observability.data_lineage.target_database IS 
'Target database name - where output data was written';

COMMENT ON COLUMN Observability.data_lineage.target_table IS 
'Target table name - output table from transformation';

COMMENT ON COLUMN Observability.data_lineage.transformation_type IS 
'Transformation type - ETL, FEATURE_ENG, AGGREGATION, JOIN, EMBEDDING_GEN, etc.';

COMMENT ON COLUMN Observability.data_lineage.transformation_logic IS 
'Transformation logic description - SQL, algorithm, or process description';

COMMENT ON COLUMN Observability.data_lineage.job_name IS 
'Job or process name - identifies the transformation job';

COMMENT ON COLUMN Observability.data_lineage.run_dts IS 
'When transformation ran - timestamp of execution';

COMMENT ON COLUMN Observability.data_lineage.openlineage_run_id IS 
'OpenLineage run UUID - correlates with OpenLineage events for external lineage systems (Marquez, Amundsen)';

COMMENT ON COLUMN Observability.data_lineage.openlineage_job_name IS 
'OpenLineage job name - job identifier in OpenLineage format';

COMMENT ON COLUMN Observability.data_lineage.openlineage_namespace IS 
'OpenLineage namespace - environment or cluster identifier (production, staging, etc.)';

COMMENT ON COLUMN Observability.data_lineage.records_read IS 
'Number of records read from source - input volume metric';

COMMENT ON COLUMN Observability.data_lineage.records_written IS 
'Number of records written to target - output volume metric';

COMMENT ON COLUMN Observability.data_lineage.run_status IS 
'Execution status - SUCCESS, FAILED, PARTIAL - indicates transformation outcome';

COMMENT ON COLUMN Observability.data_lineage.created_at IS 
'Timestamp when lineage record was created';
```

### 3.4 model_performance

```sql
CREATE TABLE Observability.model_performance (
    performance_key INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    model_id VARCHAR(100) NOT NULL,
    model_version VARCHAR(20) NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    metric_value DECIMAL(10,6),
    evaluation_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    sample_size INTEGER,
    is_sla_met BYTEINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP(6) WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP(6)
)
PRIMARY INDEX (performance_key);

COMMENT ON TABLE Observability.model_performance IS 
'Model performance metrics over time - tracks ML model accuracy, latency, and drift for monitoring';

COMMENT ON COLUMN Observability.model_performance.performance_key IS 
'Surrogate key for performance metric record';

COMMENT ON COLUMN Observability.model_performance.model_id IS 
'Model identifier - unique name of ML model being monitored';

COMMENT ON COLUMN Observability.model_performance.model_version IS 
'Model version - tracks which version of model is being evaluated';

COMMENT ON COLUMN Observability.model_performance.metric_name IS 
'Performance metric name - ACCURACY, PRECISION, RECALL, AUC, LATENCY_MS, DRIFT_SCORE, etc.';

COMMENT ON COLUMN Observability.model_performance.metric_value IS 
'Metric value - actual measured performance value';

COMMENT ON COLUMN Observability.model_performance.evaluation_dts IS 
'Evaluation timestamp - when performance was measured';

COMMENT ON COLUMN Observability.model_performance.sample_size IS 
'Evaluation sample size - number of predictions evaluated';

COMMENT ON COLUMN Observability.model_performance.is_sla_met IS 
'SLA met indicator - 1 = meets performance SLA, 0 = below threshold - enables alerting';

COMMENT ON COLUMN Observability.model_performance.created_at IS 
'Timestamp when performance record was created';
```

### 3.5 agent_outcome

```sql
CREATE TABLE Observability.agent_outcome (
    outcome_key INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    agent_id VARCHAR(100) NOT NULL,
    session_id VARCHAR(100),
    action_type VARCHAR(50) NOT NULL,
    action_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    tables_accessed VARCHAR(1000),
    outcome_status VARCHAR(20) NOT NULL,
    user_feedback VARCHAR(20),
    execution_time_ms INTEGER,
    records_processed INTEGER,
    created_at TIMESTAMP(6) WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP(6)
)
PRIMARY INDEX (outcome_key);

COMMENT ON TABLE Observability.agent_outcome IS 
'Agent action outcomes and user feedback - enables closed-loop learning by tracking what worked';

COMMENT ON COLUMN Observability.agent_outcome.outcome_key IS 
'Surrogate key for outcome record';

COMMENT ON COLUMN Observability.agent_outcome.agent_id IS 
'Agent identifier - which agent performed this action';

COMMENT ON COLUMN Observability.agent_outcome.session_id IS 
'Session identifier - links outcome to agent session context';

COMMENT ON COLUMN Observability.agent_outcome.action_type IS 
'Action type - QUERY, RECOMMENDATION, DECISION, PREDICTION - classifies agent action';

COMMENT ON COLUMN Observability.agent_outcome.action_dts IS 
'Action timestamp - when agent action was performed';

COMMENT ON COLUMN Observability.agent_outcome.tables_accessed IS 
'Tables accessed - comma-separated list of database.table names involved in action, TABLE LEVEL tracking';

COMMENT ON COLUMN Observability.agent_outcome.outcome_status IS 
'Outcome status - SUCCESS, PARTIAL, FAILED - indicates whether action achieved goal';

COMMENT ON COLUMN Observability.agent_outcome.user_feedback IS 
'User feedback - POSITIVE, NEUTRAL, NEGATIVE, CORRECTION - enables learning from human feedback';

COMMENT ON COLUMN Observability.agent_outcome.execution_time_ms IS 
'Execution time in milliseconds - performance metric for action';

COMMENT ON COLUMN Observability.agent_outcome.records_processed IS 
'Number of records processed - scale metric, aggregate count not individual keys';

COMMENT ON COLUMN Observability.agent_outcome.created_at IS 
'Timestamp when outcome record was created';
```

---

## 4. Open Standards Integration

### 4.1 OpenLineage

**Reference**: https://openlineage.io/

Store OpenLineage identifiers (run_id, job_name, namespace) to correlate with external lineage systems.

### 4.2 Data Quality Frameworks

Reference: Great Expectations, Deequ, Monte Carlo

Use standard metric names for consistency.

---

## 5. Integration with Other Modules

### 5.1 Integration with Domain (Table-Level Change Tracking)

**Pattern**: Track changes at table level with aggregate metrics

```sql
-- ETL updates 250,000 Party records
INSERT INTO Observability.change_event (
    database_name, table_name, change_type, change_dts,
    changed_by, change_source, records_affected, batch_id
) VALUES (
    'Domain', 'Party_H', 'UPDATE', CURRENT_TIMESTAMP(6),
    'ETL_DAILY', 'ETL', 250000, 'BATCH-2024-03-15'
);

-- Query change history for table
SELECT 
    change_type,
    change_dts,
    changed_by,
    records_affected,
    batch_id
FROM Observability.change_event
WHERE table_name = 'Party_H'
  AND change_dts >= CURRENT_DATE - 30
ORDER BY change_dts DESC;
```

**Benefits**: One event per batch (not millions per record), efficient audit trail

### 5.2 Feed Memory Module (Closed-Loop Learning)

**Pattern**: Observability outcomes feed Memory learnings

```sql
-- Identify successful query patterns for Memory
INSERT INTO Memory.learned_strategy (
    strategy_name, strategy_category, success_rate,
    discovered_by_agent, scope_level
)
SELECT 
    'FastPartyQueries_' || CURRENT_DATE,
    'QUERY_OPTIMIZATION',
    SUM(CASE WHEN execution_time_ms < 1000 THEN 1 ELSE 0 END) * 1.0 / COUNT(*),
    'observability_analyzer',
    'ORGANIZATION'
FROM Observability.agent_outcome
WHERE action_type = 'QUERY'
  AND tables_accessed LIKE '%Party_H%'
  AND action_dts >= CURRENT_DATE - 7
HAVING COUNT(*) >= 10;
```

### 5.3 Monitor All Modules

**Track quality and performance across all modules**:

```sql
-- Monitor quality across Domain, Prediction, Search
SELECT 
    database_name,
    table_name,
    metric_name,
    AVG(metric_value) AS avg_quality,
    MIN(metric_value) AS min_quality
FROM Observability.data_quality_metric
WHERE table_name IN ('Party_H', 'customer_features', 'product_embedding')
  AND measured_dts >= CURRENT_DATE - 7
GROUP BY database_name, table_name, metric_name;

-- Track data lineage across modules
SELECT 
    source_database || '.' || source_table AS source,
    target_database || '.' || target_table AS target,
    transformation_type,
    records_written,
    run_status
FROM Observability.data_lineage
WHERE run_dts >= CURRENT_DATE - 1
ORDER BY run_dts DESC;
```

---

## 6. Designer Responsibilities

### 6.1 Required Tables

- change_event
- data_quality_metric  
- data_lineage
- model_performance (if using ML)
- agent_outcome (if using agents)

### 6.2 Design Checklist

- [ ] Quality metrics defined
- [ ] Quality thresholds set
- [ ] OpenLineage integration configured
- [ ] Retention policies defined
- [ ] Integration with Memory configured

---

## Appendix: Quick Reference

### Core Tables
```
✅ change_event           -- Audit trail
✅ data_quality_metric    -- Quality metrics
✅ data_lineage          -- Lineage (OpenLineage)
✅ model_performance      -- Model metrics
✅ agent_outcome          -- Agent outcomes
```

### Key Principles
```
✅ Store events/metrics (NOT data)
✅ Table-level change tracking (NOT individual record changes)
✅ Aggregate metrics (records_affected count, not individual keys)
✅ Align with OpenLineage for data lineage
✅ Feed Memory module (closed-loop learning)
✅ Monitor all modules
✅ Event-scale volume acceptable (millions of events OK)
✅ Don't duplicate business data
```

### Open Standards
```
OpenLineage:         openlineage.io
Great Expectations:  Data quality
OpenTelemetry:       Observability
```
---

## Document Change Log

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-02-13 | Initial draft of Design Standard | Nathan Green, Worldwide Data Architecture Team, Teradata |
| 1.1 | 2025-02-27 | changed meets_threshold & meets_sla to is_threshold_met & is_sla_met to be consistent with booleans accross modules | Nathan Green, Worldwide Data Architecture Team, Teradata |

---

**End of Observability Module Design Standard**

