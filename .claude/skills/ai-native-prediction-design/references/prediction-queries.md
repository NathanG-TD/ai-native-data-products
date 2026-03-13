# Prediction Query Patterns Reference
## Point-in-Time Correctness, Training Datasets, Agent Discovery

---

## 1. Point-in-Time Feature Retrieval

The most critical query pattern in the Prediction module. Gets features exactly as they existed at a given moment — prevents data leakage in ML training.

### Current Features (Serving / Inference)

```sql
-- Wide format: current features for one entity
SELECT {feature_columns}, observation_dts
FROM Prediction.{entity}_{group}_features
WHERE entity_key = {entity_key}
  AND is_current = 1;

-- Tall format: current features for one entity
SELECT feature_name, value_numeric, value_type, observation_dts
FROM Prediction.feature_value
WHERE entity_key = {entity_key}
  AND entity_type = '{ENTITY_TYPE}'
  AND is_current = 1;
```

### Point-in-Time Features (Training / Backtesting)

```sql
-- "What were the features for entity {key} as at {training_date}?"
-- This is the critical data-leakage-safe pattern for ML training

-- Wide format
SELECT {feature_columns}, observation_dts
FROM Prediction.{entity}_{group}_features
WHERE entity_key = {entity_key}
  AND observation_dts <= TIMESTAMP '{training_date} 23:59:59.999999+00:00'
  AND valid_from_dts  <= TIMESTAMP '{training_date} 23:59:59.999999+00:00'
  AND valid_to_dts    >  TIMESTAMP '{training_date} 00:00:00+00:00';

-- Tall format
SELECT feature_name, value_numeric, observation_dts
FROM Prediction.feature_value
WHERE entity_key  = {entity_key}
  AND entity_type = '{ENTITY_TYPE}'
  AND observation_dts <= TIMESTAMP '{training_date} 23:59:59.999999+00:00'
  AND valid_from_dts  <= TIMESTAMP '{training_date} 23:59:59.999999+00:00'
  AND valid_to_dts    >  TIMESTAMP '{training_date} 00:00:00+00:00';
```

### Temporal Alignment — Features + Domain at Same Point in Time

Join feature state to Domain entity state at the same historical moment. Critical for training where both the label and the features must be leakage-free.

```sql
-- Features AND Domain entity state as at {training_date}
SELECT
    p.party_id,
    p.legal_name,
    p.party_type_code,          -- Domain state at training_date
    f.feature_name,
    f.value_numeric,            -- Feature value at training_date
    f.observation_dts
FROM Domain.Party_H p
INNER JOIN Prediction.feature_value f
    ON f.entity_key  = p.party_key
   AND f.entity_type = 'PARTY'
WHERE p.party_id = '{entity_id}'
  -- Domain: entity state at training_date
  AND TIMESTAMP '{training_date} 00:00:00+00:00' BETWEEN p.valid_from_dts AND p.valid_to_dts
  AND p.is_deleted = 0
  -- Features: values available at training_date
  AND f.observation_dts <= TIMESTAMP '{training_date} 23:59:59.999999+00:00'
  AND f.valid_from_dts  <= TIMESTAMP '{training_date} 23:59:59.999999+00:00'
  AND f.valid_to_dts    >  TIMESTAMP '{training_date} 00:00:00+00:00';
```

---

## 2. Training Dataset Generation

Bulk point-in-time feature extraction across all entities for a training window.

```sql
-- Generate training dataset: all entities, features as at each label_date
-- label_dates typically come from a labelled event table (churn_event, fraud_event, etc.)

WITH training_labels AS (
    -- Bring your labelled events: entity_key, label_date, label_value
    SELECT entity_key, label_date, label_value
    FROM {YourLabelTable}
    WHERE label_date BETWEEN DATE '{train_start}' AND DATE '{train_end}'
)
SELECT
    lbl.entity_key,
    lbl.label_date,
    lbl.label_value,            -- Target variable for model training
    -- Wide format features at label_date
    f.recency_score,
    f.frequency_score,
    f.monetary_score,
    -- [other feature columns]
    f.observation_dts           -- For audit / reproducibility
FROM training_labels lbl
INNER JOIN Prediction.{entity}_{group}_features f
    ON f.entity_key = lbl.entity_key
   AND f.observation_dts <= CAST(lbl.label_date AS TIMESTAMP WITH TIME ZONE)
   AND f.valid_from_dts  <= CAST(lbl.label_date AS TIMESTAMP WITH TIME ZONE)
   AND f.valid_to_dts    >  CAST(lbl.label_date AS TIMESTAMP WITH TIME ZONE)
ORDER BY lbl.entity_key, lbl.label_date;
```

---

## 3. Model Prediction Queries

```sql
-- Current prediction for an entity from a specific model
SELECT
    prediction_value,
    prediction_class,
    confidence_score,
    prediction_dts,
    feature_observation_dts
FROM Prediction.model_prediction
WHERE entity_key  = {entity_key}
  AND entity_type = '{ENTITY_TYPE}'
  AND model_id    = '{model_id}'
  AND is_current  = 1;

-- Latest prediction across all models for an entity
SELECT
    model_id,
    model_version,
    prediction_value,
    confidence_score,
    prediction_dts
FROM Prediction.model_prediction
WHERE entity_key  = {entity_key}
  AND entity_type = '{ENTITY_TYPE}'
  AND is_current  = 1
ORDER BY prediction_dts DESC;

-- Prediction with Domain entity context
SELECT
    p.party_id,
    p.legal_name,
    mp.model_id,
    mp.prediction_value,
    mp.confidence_score,
    mp.prediction_dts
FROM Prediction.model_prediction mp
INNER JOIN Domain.Party_H p
    ON p.party_key = mp.entity_key
   AND p.is_current = 1
   AND p.is_deleted = 0
WHERE mp.entity_type = 'PARTY'
  AND mp.model_id    = '{model_id}'
  AND mp.is_current  = 1
ORDER BY mp.prediction_value DESC;  -- e.g. highest churn risk first
```

---

## 4. Feature Refresh Pattern

When Domain entity data changes, dependent features should be regenerated.

```sql
-- Step 1: Expire current feature version
UPDATE Prediction.{entity}_{group}_features
SET valid_to_dts = CURRENT_TIMESTAMP(6),
    is_current   = 0
WHERE entity_key = {changed_entity_key}
  AND is_current = 1;

-- Step 2: Insert recomputed feature values
INSERT INTO Prediction.{entity}_{group}_features (
    entity_key, entity_type,
    {feature_columns},
    observation_dts, valid_from_dts, valid_to_dts, is_current,
    feature_group_name, feature_group_version,
    computation_dts, source_system, created_by
) VALUES (
    {entity_key}, '{ENTITY_TYPE}',
    {recomputed_feature_values},
    CURRENT_TIMESTAMP(6),           -- observation_dts
    CURRENT_TIMESTAMP(6),           -- valid_from_dts
    TIMESTAMP '9999-12-31 23:59:59.999999+00:00',
    1,                              -- is_current
    '{feature_group_name}',
    '{version}',
    CURRENT_TIMESTAMP(6),
    '{source_system}',
    CURRENT_USER
);
```

---

## 5. Agent Discovery Queries

Agents use Semantic to discover features, then Prediction for values.

```sql
-- Step 1: Discover available feature tables (via Semantic)
SELECT table_name, column_name, business_description
FROM Semantic.column_metadata
WHERE database_name = '{PredictionDatabase}'
  AND is_active = 'Y'
ORDER BY table_name, column_name;

-- Step 2: Understand which entities have features
SELECT DISTINCT
    entity_type,
    COUNT(DISTINCT entity_key) AS entity_cnt,
    MAX(observation_dts)       AS latest_observation,
    MIN(observation_dts)       AS earliest_observation
FROM Prediction.{entity}_{group}_features
WHERE is_current = 1
GROUP BY entity_type;

-- Step 3: Get all current features for an entity (tall format)
SELECT feature_name, feature_group, value_numeric, value_type, observation_dts
FROM Prediction.feature_value
WHERE entity_key  = {entity_key}
  AND entity_type = '{ENTITY_TYPE}'
  AND is_current  = 1
ORDER BY feature_group, feature_name;

-- Step 4: Check feature freshness relative to Domain updates
SELECT
    f.entity_key,
    f.entity_type,
    MAX(f.observation_dts)    AS latest_feature_dts,
    MAX(d.transaction_from_dts) AS latest_domain_update
FROM Prediction.{entity}_{group}_features f
INNER JOIN Domain.{Entity}_H d
    ON d.{entity}_key = f.entity_key
   AND d.is_current = 1
WHERE f.is_current = 1
GROUP BY f.entity_key, f.entity_type
HAVING MAX(d.transaction_from_dts) > MAX(f.observation_dts)
ORDER BY f.entity_key;
-- Returns entities where Domain has been updated since last feature computation
```
