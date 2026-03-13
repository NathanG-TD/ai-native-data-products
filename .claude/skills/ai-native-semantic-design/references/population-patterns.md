# Population Patterns Reference
## Seed Data Templates for AI-Native Semantic Module

Populate tables in this order: data_product_map → entity_metadata → naming_standard → column_metadata → table_relationship

---

## 1. data_product_map — Module Registry Seed

Replace `{ProductName}` with your data product name (e.g., `Customer360`).

```sql
-- SEPARATE_DB layout example
INSERT INTO Semantic.data_product_map
(module_name, module_description, module_purpose,
 database_name, naming_pattern, table_prefix,
 primary_tables, primary_views,
 module_version, deployment_status, deployed_dts, is_active)
VALUES
('Domain',
 'Core business entities and authoritative source of truth',
 'Business entity storage with full temporal history',
 '{ProductName}_Domain', 'SEPARATE_DB', NULL,
 'Party_H, Product_H, Transaction_H',
 'Party_Current, Product_Current',
 '2.0', 'DEPLOYED', CURRENT_TIMESTAMP(6), 'Y'),

('Semantic',
 'Schema metadata, relationships, and naming conventions',
 'Enable agent discovery and correct SQL generation',
 '{ProductName}_Semantic', 'SEPARATE_DB', NULL,
 'entity_metadata, table_relationship, data_product_map',
 'v_entity_catalog, v_relationship_paths',
 '2.0', 'DEPLOYED', CURRENT_TIMESTAMP(6), 'Y'),

('Prediction',
 'Engineered ML features and model outputs',
 'Feature store for model training and inference',
 '{ProductName}_Prediction', 'SEPARATE_DB', NULL,
 '{entity}_features, model_prediction',
 'v_{entity}_features_current',
 '1.0', 'PLANNED', NULL, 'Y'),

('Search',
 'Vector embeddings and similarity search indexes',
 'Enable semantic search and RAG retrieval patterns',
 '{ProductName}_Search', 'SEPARATE_DB', NULL,
 '{entity}_embedding',
 'v_{entity}_embedding_current',
 '1.0', 'PLANNED', NULL, 'Y'),

('Observability',
 'Data quality, lineage, and operational event tracking',
 'Monitor data product health and agent interactions',
 '{ProductName}_Observability', 'SEPARATE_DB', NULL,
 'change_event, data_quality_metric',
 'v_quality_summary',
 '1.0', 'PLANNED', NULL, 'Y'),

('Memory',
 'Agent session state and long-term learned preferences',
 'Persist agent context across interactions',
 '{ProductName}_Memory', 'SEPARATE_DB', NULL,
 'agent_session, agent_interaction',
 'v_active_sessions',
 '1.0', 'PLANNED', NULL, 'Y');
```

**SINGLE_DB_PREFIX alternative** (replace SEPARATE_DB entries):
```sql
-- All modules in one database — set naming_pattern and table_prefix
('Domain', '...', '...', '{ProductName}', 'SINGLE_DB_PREFIX', 'D_', 'D_Party_H, D_Product_H', 'D_Party_Current', '2.0', 'DEPLOYED', CURRENT_TIMESTAMP(6), 'Y')
```

---

## 2. entity_metadata — Table Catalog Seed

One row per table across **all modules**. Adjust entity names to match your domain.

```sql
-- Domain module entities
INSERT INTO Semantic.entity_metadata
(entity_name, entity_description, module_name, database_name, table_name, view_name,
 surrogate_key_column, natural_key_column, temporal_pattern,
 current_flag_column, deleted_flag_column, industry_standard, is_active)
VALUES
-- [Domain entities — repeat pattern for each]
('{EntityName}',
 '{Business description of what this entity represents and its scope}',
 'Domain',
 '{ProductName}_Domain',
 '{EntityName}_H',
 '{EntityName}_Current',
 '{entity_name}_key',
 '{entity_name}_id',
 'BI_TEMPORAL',                -- or TYPE_2_SCD or NONE
 'is_current',
 'is_deleted',
 '{FIBO|HL7|BIAN|CUSTOM}',
 'Y'),

-- [Semantic module entities — always include these]
('EntityMetadata',
 'Metadata catalog describing all tables across all modules',
 'Semantic', '{ProductName}_Semantic', 'entity_metadata', NULL,
 'entity_metadata_key', NULL, 'NONE', NULL, NULL, 'CUSTOM', 'Y'),

('TableRelationship',
 'Join registry enabling agent multi-hop path discovery',
 'Semantic', '{ProductName}_Semantic', 'table_relationship', NULL,
 'relationship_key', NULL, 'NONE', NULL, NULL, 'CUSTOM', 'Y'),

('DataProductMap',
 'Module registry - agent entry point for discovering deployed modules',
 'Semantic', '{ProductName}_Semantic', 'data_product_map', NULL,
 'module_key', NULL, 'NONE', NULL, NULL, 'CUSTOM', 'Y');
```

---

## 3. naming_standard — Convention Registry Seed

This seed covers the standard AI-Native naming conventions. Extend with enterprise-specific abbreviations.

```sql
INSERT INTO Semantic.naming_standard
(standard_type, standard_value, meaning, usage_guidance, applies_to, examples, is_active)
VALUES
-- Table suffixes
('SUFFIX', '_H',
 'History table - versioned entity with temporal tracking (bi-temporal or Type 2 SCD)',
 'Use for all core business entities that require change history. Multiple rows per entity represent versions.',
 'TABLE', 'Party_H, Product_H, Agreement_H', 'Y'),

('SUFFIX', '_R',
 'Reference/lookup table - controlled vocabulary and taxonomies',
 'Use for code/description lookup tables. Typically has effective_date/expiration_date for temporal validity.',
 'TABLE', 'PartyType_R, CountryCode_R, StatusCode_R', 'Y'),

('SUFFIX', '_Current',
 'View returning only current active records (is_current=1 AND is_deleted=0)',
 'Standard agent access pattern. Always query this view for operational data rather than the base _H table.',
 'VIEW', 'Party_Current, Product_Current, Agreement_Current', 'Y'),

('SUFFIX', '_Enriched',
 'View joining entity to commonly needed related data',
 'Use when agents frequently need related columns in a single query. Pre-joins reduce agent query complexity.',
 'VIEW', 'Party_Enriched, Product_Enriched', 'Y'),

-- Column suffixes
('SUFFIX', '_key',
 'Surrogate/system key - BIGINT, system-assigned, used for all inter-table joins',
 'Never exposed to end users. Use for all FK references between tables.',
 'COLUMN', 'party_key, product_key, agreement_key', 'Y'),

('SUFFIX', '_id',
 'Natural/business key - VARCHAR, from source system, used in user queries and external references',
 'The identifier users know. Can appear in reports and API responses.',
 'COLUMN', 'party_id, product_id, transaction_id', 'Y'),

('SUFFIX', '_dts',
 'Datetime stamp - TIMESTAMP(6) WITH TIME ZONE',
 'All temporal columns use this suffix. Always store in UTC.',
 'COLUMN', 'valid_from_dts, transaction_from_dts, created_dts', 'Y'),

('SUFFIX', '_code',
 'Lookup/type code - VARCHAR, references a _R reference table',
 'Short code values. Always have a corresponding _R table. Agents join to _R for descriptions.',
 'COLUMN', 'party_type_code, status_code, country_code', 'Y'),

('SUFFIX', '_amt',
 'Monetary amount - DECIMAL precision',
 'Use for all currency values. Separate column for currency_code if multi-currency.',
 'COLUMN', 'transaction_amt, balance_amt, fee_amt', 'Y'),

('SUFFIX', '_cnt',
 'Count/integer quantity',
 'Use for counts and integer measures.',
 'COLUMN', 'product_cnt, transaction_cnt', 'Y'),

-- Column prefixes
('PREFIX', 'is_',
 'Boolean flag - BYTEINT 1=true/yes, 0=false/no',
 'All boolean columns use is_ prefix for agent pattern recognition.',
 'COLUMN', 'is_current, is_deleted, is_active, is_verified', 'Y'),

-- Patterns
('PATTERN', '{entity}_key → {Entity}_H.{entity}_key',
 'Foreign key naming - column name indicates referenced entity',
 'All FK columns named {entity}_key where {entity} is the referenced table prefix. Agents infer joins from column names.',
 'COLUMN', 'party_key references Party_H, product_key references Product_H', 'Y'),

('PATTERN', 'is_current = 1 AND is_deleted = 0',
 'Standard current active record filter - applies to all _H tables',
 'Always include both filters for operational queries. Omit only for intentional historical analysis.',
 'ALL', 'SELECT * FROM Party_H WHERE is_current = 1 AND is_deleted = 0', 'Y');
```

---

## 4. table_relationship — Join Registry Seed

One row per directional join. Register the forward direction; `v_relationship_paths` handles reverse automatically.

```sql
INSERT INTO Semantic.table_relationship
(relationship_name, relationship_description,
 source_database, source_table, source_column,
 target_database, target_table, target_column,
 relationship_type, cardinality, relationship_meaning,
 is_mandatory, is_active)
VALUES
-- Pattern: Child → Parent (FK relationship)
('{Child}_To_{Parent}',
 '{Child} references {Parent} - each {child} record belongs to a {parent}',
 '{ProductName}_Domain', '{Child}_H', '{parent}_key',
 '{ProductName}_Domain', '{Parent}_H', '{parent}_key',
 'FOREIGN_KEY', 'M:1',
 'Each {child} is associated with exactly one {parent}',
 'Y', 'Y'),

-- Pattern: Associative/bridge table
('{Entity1}{Entity2}_To_{Entity1}',
 'Bridge table {Entity1}{Entity2} links to {Entity1}',
 '{ProductName}_Domain', '{Entity1}{Entity2}_H', '{entity1}_key',
 '{ProductName}_Domain', '{Entity1}_H', '{entity1}_key',
 'ASSOCIATIVE', 'M:1',
 'Each {entity1}-{entity2} association is linked to one {entity1}',
 'Y', 'Y'),

('{Entity1}{Entity2}_To_{Entity2}',
 'Bridge table {Entity1}{Entity2} links to {Entity2}',
 '{ProductName}_Domain', '{Entity1}{Entity2}_H', '{entity2}_key',
 '{ProductName}_Domain', '{Entity2}_H', '{entity2}_key',
 'ASSOCIATIVE', 'M:1',
 'Each {entity1}-{entity2} association is linked to one {entity2}',
 'Y', 'Y'),

-- Pattern: Reference data lookup
('{Entity}_Type_Lookup',
 '{Entity} type code references {EntityType} reference data',
 '{ProductName}_Domain', '{Entity}_H', '{entity}_type_code',
 '{ProductName}_Domain', '{EntityType}_R', '{entity_type}_code',
 'FOREIGN_KEY', 'M:1',
 'Decodes {entity}_type_code to a human-readable description',
 'N', 'Y'),

-- Pattern: Self-referencing hierarchy
('{Entity}_Hierarchy',
 '{Entity} parent-child hierarchy - self-referencing relationship',
 '{ProductName}_Domain', '{Entity}_H', 'parent_{entity}_key',
 '{ProductName}_Domain', '{Entity}_H', '{entity}_key',
 'HIERARCHY', 'M:1',
 'Each {entity} may have a parent {entity}, enabling hierarchical traversal',
 'N', 'Y');
```

---

## 5. column_metadata — Governance Focus Seed

Prioritise PII and sensitive columns. Not required to be exhaustive.

```sql
INSERT INTO Semantic.column_metadata
(database_name, table_name, column_name,
 business_description, is_pii, is_sensitive, data_classification,
 is_required, data_type, allowed_values_json, is_active)
VALUES
-- PII columns (always register these)
('{ProductName}_Domain', 'Party_H', 'legal_name',
 'Legal registered name of party',
 'Y', 'N', 'CONFIDENTIAL', 'Y', 'VARCHAR(200)', NULL, 'Y'),

('{ProductName}_Domain', 'Party_H', 'tax_identifier',
 'Tax identification number - encrypted at rest',
 'Y', 'Y', 'RESTRICTED', 'N', 'VARCHAR(50)', NULL, 'Y'),

-- Constrained code columns (document allowed values for agent validation)
('{ProductName}_Domain', 'Party_H', 'party_type_code',
 'Party classification type',
 'N', 'N', 'INTERNAL', 'Y', 'VARCHAR(20)',
 '["INDIVIDUAL","ORGANISATION","TRUST","GOVERNMENT"]', 'Y');
```
