# Prediction Tables & Views Reference
## DDL Templates for AI-Native Prediction Module

Replace `Prediction` with your actual database name (e.g., `Customer360_Prediction`).
Replace `Domain` with your Domain database name (e.g., `Customer360_Domain`).

**Critical**: `is_current` is `BYTEINT NOT NULL DEFAULT 1` — consistent with Domain module.

---

## 1. Wide Format Feature Table

Use when features in a group are always accessed together, the set is fixed, and values are dense.

```sql
CREATE TABLE Prediction.{entity}_{group}_features (
    feature_group_key       INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for this feature snapshot record',

    -- Entity reference (choose ONE pattern — be consistent within a module)
    entity_key              BIGINT NOT NULL
        COMMENT 'FK to Domain entity (party_key, product_key, etc.) — no content duplicated here',
    entity_type             VARCHAR(50) NOT NULL
        COMMENT 'Domain entity type: PARTY, PRODUCT — identifies which Domain table to join',
    -- Alternative typed FK (use instead of generic pattern when entity type is fixed):
    -- {entity}_key         BIGINT NOT NULL
    --     COMMENT 'FK to Domain.{Entity}_H.{entity}_key',

    -- ENGINEERED feature values (normalised to 0-1 unless documented otherwise)
    -- [Designer supplies — examples below]
    -- recency_score         DECIMAL(5,4)
    --     COMMENT 'ENGINEERED: Days since last transaction normalised [0,1]. Formula: 1.0 - (days/365). Recent = higher score.',
    -- frequency_score       DECIMAL(5,4)
    --     COMMENT 'ENGINEERED: Transaction count normalised [0,1]. Formula: count_30d / MAX(count_30d across all entities).',
    -- monetary_score        DECIMAL(5,4)
    --     COMMENT 'ENGINEERED: Total spend normalised [0,1]. Formula: spend_30d / MAX(spend_30d across all entities).',

    -- Temporal tracking (required — enables point-in-time correctness)
    observation_dts         TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When these feature values were observed or computed — critical for ML training without data leakage',
    valid_from_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When this feature version became valid',
    valid_to_dts            TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'When this feature version was superseded — 9999 = current version',
    is_current              BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Current version flag: 1 = current features, 0 = historical snapshot',

    -- Feature group metadata
    feature_group_name      VARCHAR(100) NOT NULL DEFAULT '{entity}_{group}'
        COMMENT 'Feature group identifier for management and versioning',
    feature_group_version   VARCHAR(20)
        COMMENT 'Version of feature engineering logic — enables A/B testing and rollback',
    computation_dts         TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When features were computed and inserted',

    -- Lineage
    source_system           VARCHAR(50)
        COMMENT 'System or process that computed features: ETL job, ML pipeline, etc.',
    created_by              VARCHAR(100)
        COMMENT 'User or process that created this feature record'
)
PRIMARY INDEX (feature_group_key);

COMMENT ON TABLE Prediction.{entity}_{group}_features IS
'{Entity} {group} feature group — ENGINEERED features normalised to [0,1] for ML model training.
 NO raw Domain values duplicated here; join to Domain.{Entity}_H via entity_key for raw attributes.
 Current features: is_current = 1. Point-in-time: filter on observation_dts and valid_from/to_dts.';
```

**Engineering formula examples** (put in `COMMENT ON COLUMN`):
```
Recency:     1.0 - (days_since_event / window_days)
Max-norm:    value / MAX(value) OVER (all entities)
Min-max:     (value - MIN(value)) / (MAX(value) - MIN(value))
Utilisation: numerator_amt / denominator_amt
Z-score:     (value - AVG(value)) / STDDEV(value)
```

---

## 2. Tall Format Feature Table

Use when features are accessed individually, the set evolves over time, or data is sparse.

```sql
CREATE TABLE Prediction.feature_value (
    feature_value_key       INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for feature value record',

    -- Entity reference
    entity_key              BIGINT NOT NULL
        COMMENT 'FK to Domain entity — identifies which entity this feature belongs to',
    entity_type             VARCHAR(50) NOT NULL
        COMMENT 'Domain entity type: PARTY, PRODUCT — identifies which Domain table to join',

    -- Feature identification
    feature_name            VARCHAR(100) NOT NULL
        COMMENT 'Feature identifier: recency_score, income_normalised, credit_utilisation',
    feature_group           VARCHAR(100)
        COMMENT 'Logical grouping: demographic, behavioral, risk — for filtering and management',

    -- Feature value (one column populated per row based on value_type)
    value_numeric           DECIMAL(18,4)
        COMMENT 'Numeric feature value — normalised to [0,1] range where appropriate',
    value_text              VARCHAR(500)
        COMMENT 'Categorical or text feature value',
    value_json              JSON
        COMMENT 'Complex multi-dimensional feature value stored as JSON',
    value_type              VARCHAR(20) NOT NULL
        COMMENT 'Type indicator: NUMERIC | TEXT | JSON | BOOLEAN — identifies which value column has data',

    -- Temporal tracking
    observation_dts         TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When feature was observed or computed — critical for point-in-time ML training without data leakage',
    valid_from_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When this feature version became valid',
    valid_to_dts            TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'When superseded — 9999 = current',
    is_current              BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Current version: 1 = current, 0 = historical',

    -- Metadata
    feature_version         VARCHAR(20)
        COMMENT 'Version of feature computation logic — tracks engineering changes over time',
    computation_dts         TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When feature was computed and stored',
    source_system           VARCHAR(50)
        COMMENT 'System that computed the feature: ML pipeline, ETL process, etc.',
    created_by              VARCHAR(100)
        COMMENT 'User or process that created this feature value'
)
PRIMARY INDEX (feature_value_key);

COMMENT ON TABLE Prediction.feature_value IS
'Feature values in tall format — one row per feature per entity snapshot.
 Flexible for sparse or evolving feature sets. value_type determines which value column holds data.
 All numeric values normalised to [0,1] unless documented otherwise in COMMENT ON COLUMN.
 Current: is_current = 1. Point-in-time: filter on observation_dts and valid_from/to_dts.';
```

---

## 3. Model Prediction Table

```sql
CREATE TABLE Prediction.model_prediction (
    prediction_key          INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for prediction record',

    -- Entity reference
    entity_key              BIGINT NOT NULL
        COMMENT 'FK to Domain entity — identifies which entity this prediction is about',
    entity_type             VARCHAR(50) NOT NULL
        COMMENT 'Entity type: PARTY, PRODUCT, TRANSACTION — enables polymorphic references',

    -- Model identification
    model_id                VARCHAR(100) NOT NULL
        COMMENT 'Unique model identifier: churn_classifier_v2, fraud_scorer_v1',
    model_version           VARCHAR(20) NOT NULL
        COMMENT 'Model version — tracks which version produced this prediction',

    -- Prediction output
    prediction_value        DECIMAL(10,6)
        COMMENT 'Numeric prediction: probability score, risk score, or continuous value (0.0–1.0 for probabilities)',
    prediction_class        VARCHAR(100)
        COMMENT 'Classification result: predicted class or category label',
    prediction_json         JSON
        COMMENT 'Complex prediction: multi-class probabilities or structured outputs as JSON',
    confidence_score        DECIMAL(5,4)
        COMMENT 'Model confidence in prediction — 0.0 to 1.0, higher = more confident',

    -- Temporal tracking
    prediction_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When prediction was generated by the model',
    feature_observation_dts TIMESTAMP(6) WITH TIME ZONE
        COMMENT 'When the input features were observed — links prediction to its feature state',
    valid_from_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When this prediction version became valid',
    valid_to_dts            TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'When superseded — 9999 = current prediction',
    is_current              BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Current prediction: 1 = latest, 0 = historical (superseded by newer model run)',

    -- Audit
    created_by              VARCHAR(100)
        COMMENT 'Process or system that generated prediction: model serving API, batch scoring job',
    created_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When prediction record was inserted'
)
PRIMARY INDEX (prediction_key);

COMMENT ON TABLE Prediction.model_prediction IS
'ML model prediction outputs with temporal tracking.
 Stores scores, classifications, and confidence for each entity per model version.
 Current predictions: is_current = 1. Join to Domain via entity_key + entity_type for context.
 feature_observation_dts links each prediction to the feature snapshot used as input.';
```

---

## 4. Standard Views

### `v_{entity}_features_current` — Current Features (Required)

```sql
CREATE VIEW Prediction.v_{entity}_features_current AS
SELECT
    entity_key,
    entity_type,
    -- [Include all engineered feature columns]
    observation_dts,
    feature_group_version,
    computation_dts
FROM Prediction.{entity}_{group}_features
WHERE is_current = 1;

COMMENT ON VIEW Prediction.v_{entity}_features_current IS
'Current {entity} engineered features — filters is_current=1 for active feature serving and real-time scoring.
 Engineered values only; join to Domain for raw attributes.';
```

### `v_{entity}_features_enriched` — Features + Domain Context (Required)

Provides complete ML input without duplicating Domain values in storage.

```sql
CREATE VIEW Prediction.v_{entity}_features_enriched AS
SELECT
    -- Raw attributes from Domain (NOT stored in Prediction)
    {dom}.{entity}_id,
    {dom}.{key_business_columns},   -- e.g. legal_name, birth_date, annual_income_amt

    -- Engineered features from Prediction
    cf.{feature_columns},
    cf.observation_dts,
    cf.feature_group_version

FROM Prediction.v_{entity}_features_current cf
INNER JOIN Domain.{Entity}_H {dom}
    ON {dom}.{entity}_key = cf.entity_key
   AND {dom}.is_current = 1
   AND {dom}.is_deleted = 0;

COMMENT ON VIEW Prediction.v_{entity}_features_enriched IS
'{Entity} engineered features joined with Domain raw attributes — no data duplication.
 Use for complete ML model input: engineered features (from Prediction) + raw context (from Domain).
 Always use this view rather than duplicating Domain columns into feature tables.';
```

### `v_{entity}_features_pit` — Point-in-Time Reconstruction (Required)

For training dataset generation — critical to prevent data leakage.

```sql
CREATE VIEW Prediction.v_{entity}_features_pit AS
SELECT
    {dom}.{entity}_id,
    cf.entity_key,
    -- [All feature columns]
    cf.observation_dts,
    cf.valid_from_dts,
    cf.valid_to_dts,
    -- Indicator for point-in-time filtering
    CASE WHEN cf.valid_to_dts = TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
         THEN 1 ELSE 0 END AS is_latest_version
FROM Prediction.{entity}_{group}_features cf
INNER JOIN Domain.{Entity}_H {dom}
    ON {dom}.{entity}_key = cf.entity_key
   AND {dom}.is_current = 1;  -- Join current Domain record to historical features

COMMENT ON VIEW Prediction.v_{entity}_features_pit IS
'{Entity} point-in-time feature history — all versions for training dataset generation.
 Filter by observation_dts and valid_from/to_dts to reconstruct features at any historical moment.
 Do NOT filter is_current=1 here — training requires all historical snapshots.
 Use valid_from_dts <= training_date < valid_to_dts for correct temporal reconstruction.';
```

---

## 5. Anti-Pattern Reference (What NOT to Do)

```sql
-- ❌ WRONG: Raw Domain value duplicated in feature table
CREATE TABLE Prediction.customer_features_BAD (
    party_key        BIGINT,
    legal_name       VARCHAR(200),    -- ❌ already in Domain.Party_H
    birth_date       DATE,            -- ❌ already in Domain.Party_H
    annual_income    DECIMAL(15,2),   -- ❌ raw value, not engineered
    credit_limit     DECIMAL(15,2),   -- ❌ raw value, not engineered
    age_normalised   DECIMAL(5,4)     -- ✅ engineered
);

-- ✅ CORRECT: Only engineered features, raw values joined via view
CREATE TABLE Prediction.customer_features_GOOD (
    party_key           BIGINT,
    age_normalised      DECIMAL(5,4),  -- ✅ (birth_date → age → normalised)
    income_normalised   DECIMAL(5,4),  -- ✅ (income → income / max_income)
    credit_utilisation  DECIMAL(5,4),  -- ✅ (balance / credit_limit)
    recency_score       DECIMAL(5,4),  -- ✅ (days_since_last_tx → normalised)
    is_current          BYTEINT NOT NULL DEFAULT 1
);
-- Raw values (legal_name, birth_date, annual_income) accessed via v_customer_features_enriched
```
