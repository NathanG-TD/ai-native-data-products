# Domain Structure Reference
## DDL Templates for AI-Native Domain Module

---

## 1. Core Entity Table Template (`{Entity}_H`)

```sql
CREATE TABLE {EntityName}_H (
    -- Identity (Required)
    {entity}_key        BIGINT NOT NULL
        COMMENT 'Surrogate key - system-assigned, never reused, used for all module joins',
    {entity}_id         VARCHAR(50) NOT NULL
        COMMENT 'Natural/business key from source system - used in user queries and external references',

    -- Bi-Temporal (Advocated - see implementation-guidance.md for alternatives)
    valid_from_dts      TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When this version became true in the real world (valid time start)',
    valid_to_dts        TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'When this version stopped being true in real world - 9999 = currently valid',
    transaction_from_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this version was inserted into the database (transaction time start)',
    transaction_to_dts  TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'When this version was superseded - 9999 = current in database',

    -- State flags (Required)
    is_current          BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Current version indicator: 1 = current, 0 = historical. Filter: is_current = 1',
    is_deleted          BYTEINT NOT NULL DEFAULT 0
        COMMENT 'Soft delete: 1 = logically deleted but retained for audit, 0 = active',

    -- [Designer-supplied business attributes]
    -- {attribute}       {DATA_TYPE}
    --     COMMENT '{Business meaning, context, constraints, source reference}',

    -- Audit (choose: Core only vs Extended - see implementation-guidance.md)
    created_by_user_id  VARCHAR(50) NOT NULL DEFAULT CURRENT_USER
        COMMENT 'Database user who created this record version',
    updated_by_user_id  VARCHAR(50) NOT NULL DEFAULT CURRENT_USER
        COMMENT 'Database user who last updated this record version',
    source_system_id    VARCHAR(50)
        COMMENT 'Source system that originated this data - e.g. CRM, ERP, MDM',
    batch_id            BIGINT
        COMMENT 'ETL/ELT batch identifier for load traceability'
)
PRIMARY INDEX ({entity}_key);   -- NUPI: allows multiple versions of same key

-- Table comment (Required)
COMMENT ON TABLE {EntityName}_H IS
'{EntityName} entity history - authoritative source of truth for {entity} with full bi-temporal versioning.
 Current records: is_current = 1 AND is_deleted = 0. Use {EntityName}_Current view for simplified access.';

-- UNIQUE INDEX for natural key lookup on current records
CREATE UNIQUE INDEX idx_{entity}_natural_key_current
ON {EntityName}_H ({entity}_id)
WHERE is_current = 1 AND is_deleted = 0;
```

---

## 2. Reference Data Table Template (`{Reference}_R`)

```sql
CREATE TABLE {ReferenceName}_R (
    {reference}_key     BIGINT NOT NULL
        COMMENT 'Surrogate key for reference data entry',
    {reference}_code    VARCHAR(20) NOT NULL
        COMMENT 'Reference code - short identifier used in domain tables, unique within effective period',
    short_description   VARCHAR(100) NOT NULL
        COMMENT 'Brief label for UI dropdowns and reports',
    long_description    VARCHAR(500)
        COMMENT 'Complete definition and usage guidance for this reference value',

    -- Temporal (simple date range - no bi-temporal needed for reference data)
    effective_date      DATE NOT NULL
        COMMENT 'Date this reference value becomes valid for use',
    expiration_date     DATE NOT NULL DEFAULT DATE '9999-12-31'
        COMMENT 'Date this value expires - 9999-12-31 = indefinitely valid',
    is_current          BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Current validity: 1 = currently valid, 0 = expired or not yet effective',

    -- Optional hierarchy
    parent_{reference}_key BIGINT
        COMMENT 'Parent reference key for hierarchical taxonomies - NULL for top-level values',
    sort_order          INTEGER
        COMMENT 'Display sequence for ordering in UI or reports'
)
UNIQUE PRIMARY INDEX ({reference}_code, effective_date);
-- UNIQUE PI valid here: reference codes are unique per effective period

COMMENT ON TABLE {ReferenceName}_R IS
'{ReferenceName} reference data - controlled vocabulary for {reference}_code columns.
 Current values: is_current = 1. Join on {entity}.{reference}_code = ref.{reference}_code.';
```

---

## 3. Relationship Table Template (`{Entity1}{Entity2}_H`)

```sql
CREATE TABLE {Entity1}{Entity2}_H (
    {entity1}_{entity2}_key BIGINT NOT NULL
        COMMENT 'Surrogate key for this relationship instance',
    {entity1}_key       BIGINT NOT NULL
        COMMENT 'FK to {Entity1}_H.{entity1}_key - identifies the first entity in relationship',
    {entity2}_key       BIGINT NOT NULL
        COMMENT 'FK to {Entity2}_H.{entity2}_key - identifies the second entity in relationship',

    -- Same temporal pattern as entity tables (consistency for agent pattern learning)
    valid_from_dts      TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When this relationship became active in real world',
    valid_to_dts        TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'When this relationship ended in real world - 9999 = currently active',
    transaction_from_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this relationship version was recorded',
    transaction_to_dts  TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'When superseded in database - 9999 = current',
    is_current          BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Current version: 1 = current active relationship, 0 = historical',
    is_deleted          BYTEINT NOT NULL DEFAULT 0
        COMMENT 'Soft delete: 1 = terminated but retained, 0 = active',

    -- [Designer-supplied relationship attributes]
    -- role_type_code    VARCHAR(20)    -- Contextual type of relationship
    -- effective_pct     DECIMAL(5,2)   -- Relationship weight or percentage (if applicable)

    PRIMARY INDEX ({entity1}_key)  -- Co-locate with first entity for join efficiency
);

COMMENT ON TABLE {Entity1}{Entity2}_H IS
'{Entity1} to {Entity2} association history - many-to-many relationship with temporal tracking.
 Current active: is_current = 1 AND is_deleted = 0. Primary join key: {entity1}_key.';
```

---

## 4. Standard Views

### `{Entity}_Current` (Required for every entity)

```sql
CREATE VIEW {Entity}_Current AS
SELECT {entity}_key, {entity}_id,
       -- [Include all business-relevant columns; exclude internal temporal columns]
       valid_from_dts, transaction_from_dts,
       source_system_id
FROM {Entity}_H
WHERE is_current = 1 AND is_deleted = 0;

COMMENT ON VIEW {Entity}_Current IS
'Current active {entity} records - filters is_current=1 AND is_deleted=0.
 Use this view as default access pattern; join to other modules via {entity}_key.';
```

### `{Entity}_Enriched` (Optional — for frequent join patterns)

```sql
CREATE VIEW {Entity}_Enriched AS
SELECT
    e.*,
    ref.short_description   AS {type}_description   -- Example: join to reference data
    -- [Add other frequent joins here]
FROM {Entity}_Current e
LEFT JOIN {TypeReference}_R ref
    ON e.{type}_code = ref.{type}_code
   AND ref.is_current = 1;

COMMENT ON VIEW {Entity}_Enriched IS
'{Entity} with decoded reference values and common related data pre-joined.
 Use when full entity context is needed in a single query.';
```

### `{Entity}_AuditTrail` (Optional — full history)

```sql
CREATE VIEW {Entity}_AuditTrail AS
SELECT *,
    CASE WHEN is_deleted = 1 THEN 'DELETED'
         WHEN is_current = 1 THEN 'CURRENT'
         ELSE 'HISTORICAL'
    END AS record_status
FROM {Entity}_H
ORDER BY {entity}_key, valid_from_dts, transaction_from_dts;

COMMENT ON VIEW {Entity}_AuditTrail IS
'Complete version history for {entity} including all superseded and deleted records.
 Do not use for operational queries - use {Entity}_Current instead.';
```

---

## 5. Cross-Module Integration Patterns

### Pattern A — Generic Reference (use when module references many entity types)

```sql
CREATE TABLE {Module}.{Table} (
    {table}_key         BIGINT NOT NULL,
    entity_key          BIGINT NOT NULL
        COMMENT 'FK to Domain entity identified by entity_type',
    entity_type         VARCHAR(50) NOT NULL
        COMMENT 'Domain entity type: PARTY, PRODUCT, AGREEMENT, etc.',
    ...
);

-- Join:
SELECT t.*, p.party_id, p.legal_name
FROM {Module}.{Table} t
INNER JOIN Domain.Party_H p
    ON p.party_key = t.entity_key AND t.entity_type = 'PARTY'
WHERE p.is_current = 1 AND p.is_deleted = 0;
```

### Pattern B — Specific References (use when module references known entity types)

```sql
CREATE TABLE {Module}.{Table} (
    {table}_key         BIGINT NOT NULL,
    party_key           BIGINT
        COMMENT 'FK to Domain.Party_H.party_key - NULL when record is not party-related',
    product_key         BIGINT
        COMMENT 'FK to Domain.Product_H.product_key - NULL when record is not product-related',
    ...
);
```

**Rule**: Choose ONE pattern per module; apply consistently. Document the choice in the Semantic module.

---

## 6. Metadata Discovery Queries (for Semantic Module)

Agents use these to self-discover the Domain module:

```sql
-- Discover all entities
SELECT TableName, CommentString FROM DBC.TablesV
WHERE DatabaseName = '{DatabaseName}' ORDER BY TableName;

-- Understand entity structure
SELECT ColumnName, ColumnType, CommentString FROM DBC.ColumnsV
WHERE DatabaseName = '{DatabaseName}' AND TableName = '{Entity}_H'
ORDER BY ColumnId;

-- Find all views available
SELECT TableName, CommentString FROM DBC.TablesV
WHERE DatabaseName = '{DatabaseName}' AND TableKind = 'V'
ORDER BY TableName;
```

These patterns should be documented in the Semantic module's `EntityMetadata` and `NamingStandards` tables so agents can reference them programmatically.
