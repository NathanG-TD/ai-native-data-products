# Implementation Guidance Reference
## Advocated Data Management Practices for AI-Native Domain Module

---

## 1. Bi-Temporal DML Patterns

### Insert (Version 1 — new entity)

```sql
INSERT INTO Party_H (
    party_key, party_id, legal_name,
    valid_from_dts, valid_to_dts,
    transaction_from_dts, transaction_to_dts,
    is_current, is_deleted
) VALUES (
    NEXT VALUE FOR seq_party_key,
    'CUST-12345', 'Jane Doe',
    TIMESTAMP '2024-01-01 00:00:00+00:00',
    TIMESTAMP '9999-12-31 23:59:59.999999+00:00',
    CURRENT_TIMESTAMP(6),
    TIMESTAMP '9999-12-31 23:59:59.999999+00:00',
    1, 0
);
```

### Update (Create Version 2 — attribute change)

```sql
-- Step 1: Close current version
UPDATE Party_H
SET valid_to_dts        = TIMESTAMP '2024-06-15 00:00:00+00:00',
    transaction_to_dts  = CURRENT_TIMESTAMP(6),
    is_current          = 0
WHERE party_key = 1001 AND is_current = 1;

-- Step 2: Insert new version (only changed + unchanged attributes)
INSERT INTO Party_H (party_key, party_id, legal_name,
    valid_from_dts, valid_to_dts, transaction_from_dts, transaction_to_dts,
    is_current, is_deleted)
VALUES (1001, 'CUST-12345', 'Jane Smith',         -- Name changed
    TIMESTAMP '2024-06-15 00:00:00+00:00',        -- When change occurred
    TIMESTAMP '9999-12-31 23:59:59.999999+00:00',
    CURRENT_TIMESTAMP(6),                          -- When we recorded it
    TIMESTAMP '9999-12-31 23:59:59.999999+00:00', 1, 0);
```

### Correct Historical Data (Late-Arriving Fact)

```sql
-- Scenario: discovered event was on 2024-06-10, not 2024-06-15

-- Step 1: Close the incorrect version in transaction time only
UPDATE Party_H
SET transaction_to_dts = CURRENT_TIMESTAMP(6), is_current = 0
WHERE party_key = 1001
  AND valid_from_dts = TIMESTAMP '2024-06-15 00:00:00+00:00'
  AND is_current = 1;

-- Step 2: Insert corrected version with new valid_from_dts
INSERT INTO Party_H (party_key, party_id, legal_name,
    valid_from_dts, valid_to_dts, transaction_from_dts, transaction_to_dts,
    is_current, is_deleted)
VALUES (1001, 'CUST-12345', 'Jane Smith',
    TIMESTAMP '2024-06-10 00:00:00+00:00',        -- CORRECTED valid time
    TIMESTAMP '9999-12-31 23:59:59.999999+00:00',
    CURRENT_TIMESTAMP(6),                          -- New transaction time
    TIMESTAMP '9999-12-31 23:59:59.999999+00:00', 1, 0);
-- Previous "incorrect" record preserved in transaction history
```

### Soft Delete

```sql
UPDATE Party_H
SET is_deleted         = 1,
    is_current         = 0,
    valid_to_dts       = CURRENT_TIMESTAMP(6),
    transaction_to_dts = CURRENT_TIMESTAMP(6),
    updated_by_user_id = CURRENT_USER
WHERE party_key = 1001 AND is_current = 1;
-- Record retained; excluded by standard filter: is_current=1 AND is_deleted=0
```

---

## 2. Bi-Temporal Query Patterns

### Current State

```sql
SELECT party_key, party_id, legal_name
FROM Party_H
WHERE party_id = 'CUST-12345'
  AND is_current = 1 AND is_deleted = 0;
```

### Point-in-Time (for ML feature computation — prevents data leakage)

```sql
-- "What was the customer's name on 2024-05-01?"
SELECT party_key, party_id, legal_name
FROM Party_H
WHERE party_id = 'CUST-12345'
  AND TIMESTAMP '2024-05-01 00:00:00+00:00' BETWEEN valid_from_dts AND valid_to_dts
  AND is_deleted = 0
  AND transaction_from_dts = (
      SELECT MAX(p2.transaction_from_dts)
      FROM Party_H p2
      WHERE p2.party_key = Party_H.party_key
        AND p2.transaction_from_dts <= TIMESTAMP '2024-05-01 23:59:59.999999+00:00'
  );
```

### As-Of Query (what did the DB contain at a specific recording time)

```sql
-- "What did our DB show on 2024-07-15?"
SELECT party_key, party_id, legal_name
FROM Party_H
WHERE party_id = 'CUST-12345'
  AND TIMESTAMP '2024-07-15 00:00:00+00:00'
      BETWEEN transaction_from_dts AND transaction_to_dts
  AND is_deleted = 0;
```

---

## 3. Surrogate Key Strategy

**Advocated**: Teradata sequence objects (portable, gap-free under normal conditions)

```sql
-- Create sequence for each entity
CREATE SEQUENCE seq_{entity}_key
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 9223372036854775807   -- BIGINT max
    NO CYCLE
    CACHE 100;

-- Usage in INSERT
NEXT VALUE FOR seq_{entity}_key

-- Cross-module: generate surrogate keys in target module too
CREATE SEQUENCE seq_{module}_{entity}_key ...
```

**Alternatives**:
- IDENTITY columns (simpler, Teradata-native)
- Hash of natural key (deterministic but collision risk)
- UUID (globally unique but 16 bytes vs 8 for BIGINT)

---

## 4. Column Optimization Strategy

Three-tier approach to balance query performance vs storage:

### Tier 1 — Core (always include)

```
{entity}_key, {entity}_id
Temporal columns (valid_from_dts, valid_to_dts, transaction_from_dts, transaction_to_dts)
is_current, is_deleted
Key business attributes (what makes this entity useful)
created_by_user_id, source_system_id
```

### Tier 2 — FK References (add for > 10M rows, replace embedded metadata)

```
change_event_key    BIGINT  -- FK to Observability.ChangeEvent_H
lineage_key         BIGINT  -- FK to Observability.DataLineage_H
quality_key         BIGINT  -- FK to Observability.QualityAssessment_H
```
Store full audit detail in Observability; use FK for access. Reduces Domain table width.

### Tier 3 — Extended Metadata (only for < 10M rows or specific needs)

```
record_hash         CHAR(64)    -- For change detection (SHA-256 of business columns)
load_sequence_nbr   BIGINT      -- Within-batch ordering
source_record_id    VARCHAR(100) -- Source system PK
extraction_dts      TIMESTAMP(6) -- When extracted from source
```

---

## 5. Physical Design Patterns

### Primary Index Selection

| Table Type | Recommended PI | Rationale |
|------------|---------------|-----------|
| Temporal entity (`_H`) | `NUPI ({entity}_key)` | Multiple versions share key |
| Reference data (`_R`) | `UPI ({ref}_code, effective_date)` | Unique per effective period |
| Relationship (`E1E2_H`) | `NUPI ({entity1}_key)` | Co-locate with primary entity |
| Current-only | `UPI ({entity}_key)` | One row per entity |

### Secondary Indexes (selective — only when query is frequent and unacceptable without it)

```sql
-- Natural key lookup (when PI is surrogate)
CREATE UNIQUE INDEX idx_{entity}_id_current
ON {Entity}_H ({entity}_id)
WHERE is_current = 1 AND is_deleted = 0;

-- FK join optimization
CREATE INDEX idx_{rel}_fk_{entity2}
ON {Entity1}{Entity2}_H ({entity2}_key)
WHERE is_current = 1 AND is_deleted = 0;

-- Composite filter
CREATE INDEX idx_{entity}_type_status
ON {Entity}_H ({type}_code, is_current, is_deleted);
```

### Join Indexes (for expensive frequently-used patterns)

```sql
-- Materialized current view (avoid if table changes frequently)
CREATE JOIN INDEX jidx_{entity}_current AS
SELECT {entity}_key, {entity}_id, {key_business_columns}
FROM {Entity}_H
WHERE is_current = 1 AND is_deleted = 0
  AND transaction_to_dts = TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
PRIMARY INDEX ({entity}_key);
```

### Statistics Collection

```sql
COLLECT STATISTICS
COLUMN ({entity}_key),
COLUMN ({entity}_id),
COLUMN ({type}_code),
COLUMN (is_current),
COLUMN (is_deleted),
COLUMN ({type}_code, is_current, is_deleted),   -- composite filter
COLUMN (valid_from_dts),
COLUMN (transaction_from_dts)
ON {Entity}_H;
```

| Table Size | Frequency | Method |
|------------|-----------|--------|
| < 1M rows | After major loads | Full scan |
| 1M–100M rows | Daily | 10% sample |
| > 100M rows | Weekly | 5% sample |
| Reference tables | After changes | Full scan |

### Compression (large text columns only)

```sql
-- AUTO COMPRESS for sparse/large text columns
CREATE TABLE {Entity}_H (...)
WITH COLUMN_PARTITION = (
    AUTO COMPRESS (notes_txt, address_text, json_attributes)
);
```

---

## 6. Timezone Handling

**Advocated**: `TIMESTAMP(6) WITH TIME ZONE` everywhere.

```sql
-- All timestamps with timezone
valid_from_dts  TIMESTAMP(6) WITH TIME ZONE

-- Store in UTC, display in local timezone at application layer
-- Avoids DST ambiguity; enables global deployments
-- Consistent comparison across time zones

-- Session timezone setting
SET TIME ZONE 'Australia/Canberra';
```

---

## 7. Data Quality Patterns

### Domain-Level Quality Columns (Core Tier only — for small tables)

```sql
data_quality_score  DECIMAL(3,2)
    COMMENT 'Quality score 0.00–1.00 computed at load time by DQ rules',
quality_issues_json JSON
    COMMENT 'Array of quality issue codes detected; NULL = no issues',
is_validated        BYTEINT NOT NULL DEFAULT 0
    COMMENT '1 = passed all DQ rules, 0 = not yet validated or has issues'
```

### Observability Offload (Advocated for > 10M rows)

Store all quality detail in `Observability.DataQualityAssessment_O`. Reference via `quality_key` FK in Domain table. Use view to surface quality status:

```sql
CREATE VIEW {Entity}_WithQuality AS
SELECT e.*, q.quality_score, q.issue_count
FROM {Entity}_Current e
LEFT JOIN Observability.DataQualityAssessment_O q
    ON q.entity_key = e.{entity}_key
   AND q.entity_type = '{ENTITY}'
   AND q.is_latest = 1;
```
