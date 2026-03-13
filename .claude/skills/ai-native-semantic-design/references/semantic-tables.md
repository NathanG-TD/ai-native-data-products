# Semantic Tables Reference
## DDL Templates for AI-Native Semantic Module

All tables use `GENERATED ALWAYS AS IDENTITY` surrogate keys.
Replace `Semantic` with your actual database name (e.g., `Customer360_Semantic`).

---

## 1. data_product_map — Module Registry

```sql
CREATE TABLE Semantic.data_product_map (
    module_key          INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    module_name         VARCHAR(50) NOT NULL
        COMMENT 'Module name: Domain, Semantic, Prediction, Search, Memory, Observability',
    module_description  VARCHAR(1000)
        COMMENT 'Business description of module purpose and scope',
    module_purpose      VARCHAR(500)
        COMMENT 'Concise statement of module purpose',
    database_name       VARCHAR(100) NOT NULL
        COMMENT 'Physical Teradata database where module is deployed - critical for agent discovery',
    naming_pattern      VARCHAR(20)
        COMMENT 'Layout: SEPARATE_DB = one database per module | SINGLE_DB_PREFIX = all modules in one DB with prefixes',
    table_prefix        VARCHAR(10)
        COMMENT 'Table name prefix if SINGLE_DB_PREFIX - e.g. D_, P_, S_; NULL for SEPARATE_DB',
    primary_tables      VARCHAR(500)
        COMMENT 'Comma-separated key table names - agent entry points for exploration',
    primary_views       VARCHAR(500)
        COMMENT 'Comma-separated key view names - common agent access patterns',
    module_version      VARCHAR(20)
        COMMENT 'Version of module design standard used',
    deployment_status   VARCHAR(20)
        COMMENT 'DEPLOYED | PLANNED | DEPRECATED',
    deployed_dts        TIMESTAMP(6) WITH TIME ZONE
        COMMENT 'When module was deployed to production',
    is_active           CHAR(1) NOT NULL DEFAULT 'Y'
        COMMENT 'Y = active module, N = deprecated',
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this registry record was created',
    updated_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When this registry record was last updated'
)
PRIMARY INDEX (module_key);

COMMENT ON TABLE Semantic.data_product_map IS
'Module registry - agents discover deployed modules and their physical Teradata database locations.
 Start here to understand what is available before querying entity_metadata.';
```

---

## 2. entity_metadata — Table Catalog

```sql
CREATE TABLE Semantic.entity_metadata (
    entity_metadata_key INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    entity_name         VARCHAR(100) NOT NULL
        COMMENT 'Business name of entity: Party, Product, Transaction (not the table name)',
    entity_description  VARCHAR(1000) NOT NULL
        COMMENT 'Business description of entity purpose, scope, and typical usage',
    module_name         VARCHAR(50) NOT NULL
        COMMENT 'Module where entity resides: Domain, Prediction, Search, Memory, Observability',
    database_name       VARCHAR(100)
        COMMENT 'Physical database name - matches data_product_map.database_name',
    table_name          VARCHAR(100) NOT NULL
        COMMENT 'Physical table name: Party_H, Product_H, customer_features',
    view_name           VARCHAR(100)
        COMMENT 'Standard current view for agent access: Party_Current, Product_Current',
    surrogate_key_column VARCHAR(100)
        COMMENT 'Surrogate key column name: party_key, product_key',
    natural_key_column  VARCHAR(100)
        COMMENT 'Natural business key column: party_id, product_id',
    temporal_pattern    VARCHAR(50)
        COMMENT 'Temporal tracking: BI_TEMPORAL | TYPE_2_SCD | NONE',
    current_flag_column VARCHAR(100)
        COMMENT 'Current version flag column name - typically is_current',
    deleted_flag_column VARCHAR(100)
        COMMENT 'Soft delete flag column name - typically is_deleted',
    industry_standard   VARCHAR(50)
        COMMENT 'Industry model standard applied: FIBO, HL7, BIAN, GS1, CUSTOM',
    is_active           CHAR(1) NOT NULL DEFAULT 'Y'
        COMMENT 'Y = active entity, N = deprecated',
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When metadata record was created',
    updated_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When metadata record was last updated'
)
PRIMARY INDEX (entity_metadata_key);

COMMENT ON TABLE Semantic.entity_metadata IS
'Entity (table) catalog - one row per table across all modules.
 Agents query here to discover what entities exist, where they live, and how to access current records.
 Use view_name for agent queries; use table_name for direct table access.';
```

---

## 3. column_metadata — Attribute Catalog

```sql
CREATE TABLE Semantic.column_metadata (
    column_metadata_key INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    database_name       VARCHAR(100) NOT NULL
        COMMENT 'Physical database name where table resides',
    table_name          VARCHAR(100) NOT NULL
        COMMENT 'Physical table name containing this column',
    column_name         VARCHAR(100) NOT NULL
        COMMENT 'Physical column name',
    business_description VARCHAR(1000)
        COMMENT 'Business meaning of this column - supplements COMMENT ON COLUMN in DDL',
    is_pii              CHAR(1) NOT NULL DEFAULT 'N'
        COMMENT 'Y = contains Personally Identifiable Information - governs privacy controls',
    is_sensitive        CHAR(1) NOT NULL DEFAULT 'N'
        COMMENT 'Y = sensitive data (SSN, credit card, health) - governs security controls',
    data_classification VARCHAR(50)
        COMMENT 'Classification level: PUBLIC | INTERNAL | CONFIDENTIAL | RESTRICTED',
    is_required         CHAR(1)
        COMMENT 'Y = NOT NULL constraint, N = nullable',
    data_type           VARCHAR(100)
        COMMENT 'Physical data type: VARCHAR, BIGINT, DECIMAL, DATE, TIMESTAMP, JSON',
    allowed_values_json JSON
        COMMENT 'JSON array of valid values for constrained columns; NULL = unconstrained',
    is_active           CHAR(1) NOT NULL DEFAULT 'Y'
        COMMENT 'Y = column active, N = deprecated or removed',
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When metadata record was created',
    updated_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When metadata record was last updated'
)
PRIMARY INDEX (column_metadata_key);

COMMENT ON TABLE Semantic.column_metadata IS
'Column (attribute) metadata - classification, sensitivity, and validation rules for each column.
 Prioritise PII/sensitive/classification flags; business_description supplements DDL COMMENTs.
 Not required to be exhaustive - focus on columns needing governance metadata.';
```

---

## 4. naming_standard — Naming Convention Registry

```sql
CREATE TABLE Semantic.naming_standard (
    naming_standard_key INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    standard_type       VARCHAR(50) NOT NULL
        COMMENT 'Convention type: SUFFIX | PREFIX | PATTERN | ABBREVIATION',
    standard_value      VARCHAR(100) NOT NULL
        COMMENT 'The naming element: _H, _key, is_, dts, _R, _Current',
    meaning             VARCHAR(500) NOT NULL
        COMMENT 'What this element means: _H = history table with temporal versioning',
    usage_guidance      VARCHAR(1000)
        COMMENT 'When and how to apply this convention',
    applies_to          VARCHAR(50)
        COMMENT 'Scope: TABLE | COLUMN | VIEW | ALL',
    examples            VARCHAR(1000)
        COMMENT 'Concrete examples: Party_H, product_key, is_current, valid_from_dts',
    is_active           CHAR(1) NOT NULL DEFAULT 'Y'
        COMMENT 'Y = convention in use, N = deprecated',
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When convention was documented'
)
PRIMARY INDEX (naming_standard_key);

COMMENT ON TABLE Semantic.naming_standard IS
'Naming convention registry - agents query here to interpret unfamiliar table and column names.
 Covers suffixes, prefixes, patterns, and abbreviations used across the data product.';
```

---

## 5. table_relationship — Join Registry

```sql
CREATE TABLE Semantic.table_relationship (
    relationship_key    INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    relationship_name   VARCHAR(100) NOT NULL
        COMMENT 'Descriptive name: PartyAddress_To_Party, Party_Hierarchy, PartyProduct_Bridge',
    relationship_description VARCHAR(1000)
        COMMENT 'Business description: what this association represents',
    source_database     VARCHAR(100)
        COMMENT 'Database containing source table (the table with the FK column)',
    source_table        VARCHAR(100) NOT NULL
        COMMENT 'Source table name - contains the foreign key column',
    source_column       VARCHAR(100) NOT NULL
        COMMENT 'Foreign key column in source table',
    target_database     VARCHAR(100)
        COMMENT 'Database containing target table (the referenced table)',
    target_table        VARCHAR(100) NOT NULL
        COMMENT 'Target table name - the table being referenced',
    target_column       VARCHAR(100) NOT NULL
        COMMENT 'Primary/unique key column in target table',
    relationship_type   VARCHAR(50) NOT NULL
        COMMENT 'FOREIGN_KEY | HIERARCHY | ASSOCIATIVE',
    cardinality         VARCHAR(20)
        COMMENT '1:1 | 1:M | M:1 | M:M',
    relationship_meaning VARCHAR(500)
        COMMENT 'Business meaning: a Party can have many Addresses over time',
    is_mandatory        CHAR(1) NOT NULL DEFAULT 'N'
        COMMENT 'Y = FK is NOT NULL (required join), N = nullable (optional join)',
    is_active           CHAR(1) NOT NULL DEFAULT 'Y'
        COMMENT 'Y = relationship valid, N = deprecated',
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When relationship was registered',
    updated_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When relationship was last updated'
)
PRIMARY INDEX (relationship_key);

COMMENT ON TABLE Semantic.table_relationship IS
'Join registry - one row per directional relationship between tables.
 Source table contains the FK column; target table contains the referenced key.
 Register FORWARD direction only - v_relationship_paths handles bidirectional traversal automatically.
 This table is the foundation for multi-hop path discovery.';
```

---

## 6. Optional Tables

### ontology (add when formal taxonomies needed)

```sql
CREATE TABLE Semantic.ontology (
    ontology_key        INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    concept_name        VARCHAR(200) NOT NULL
        COMMENT 'Ontology concept name: FIBO Party, HL7 Patient',
    concept_description VARCHAR(1000)
        COMMENT 'Concept definition from source standard',
    standard_source     VARCHAR(100)
        COMMENT 'Standard origin: FIBO, HL7, BIAN, SKOS, Schema.org, CUSTOM',
    parent_concept      VARCHAR(200)
        COMMENT 'Parent concept for hierarchical taxonomies',
    mapped_table        VARCHAR(100)
        COMMENT 'Physical table this concept maps to',
    mapped_column       VARCHAR(100)
        COMMENT 'Physical column this concept maps to (if attribute-level)',
    is_active           CHAR(1) NOT NULL DEFAULT 'Y',
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
)
PRIMARY INDEX (ontology_key);

COMMENT ON TABLE Semantic.ontology IS
'Ontology and taxonomy definitions - maps physical tables/columns to formal industry concepts.
 Use when aligning to FIBO, HL7, BIAN or other industry standards.';
```

### business_rule (add when validation rules must be queryable)

```sql
CREATE TABLE Semantic.business_rule (
    rule_key            INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY,
    rule_name           VARCHAR(100) NOT NULL
        COMMENT 'Short rule identifier: PARTY_STATUS_VALID, BALANCE_NON_NEGATIVE',
    rule_description    VARCHAR(1000) NOT NULL
        COMMENT 'Plain English description of the rule',
    applies_to_table    VARCHAR(100)
        COMMENT 'Table this rule governs',
    applies_to_column   VARCHAR(100)
        COMMENT 'Column this rule governs (NULL = table-level rule)',
    rule_expression     VARCHAR(2000)
        COMMENT 'SQL expression or pattern: status_code IN (''ACTIVE'',''INACTIVE'',''PENDING'')',
    rule_type           VARCHAR(50)
        COMMENT 'CONSTRAINT | VALIDATION | DERIVATION | BUSINESS_LOGIC',
    is_active           CHAR(1) NOT NULL DEFAULT 'Y',
    created_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
)
PRIMARY INDEX (rule_key);

COMMENT ON TABLE Semantic.business_rule IS
'Business rules registry - agents query to validate data or understand constraints.
 Include rules that are not expressible as simple NOT NULL or CHECK constraints.';
```
