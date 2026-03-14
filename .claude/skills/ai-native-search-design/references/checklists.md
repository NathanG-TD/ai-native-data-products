# Design Review Checklists
## AI-Native Search Module

---

## 1. Designer Requirements Checklist

Before writing DDL, confirm:

- [ ] Teradata version verified — VECTOR datatype requires v20.00.26.XX+
- [ ] Entities to embed identified (Party, Product, Document, …)
- [ ] Source column(s) per entity identified (description, notes, combined_text, …)
- [ ] Embedding model and output dimensions agreed
- [ ] Expected vector count per entity estimated (drives indexing decision)
- [ ] Search latency requirement: batch vs real-time
- [ ] Update strategy defined: on insert, daily batch, on-demand, change-triggered
- [ ] Table pattern chosen: generic `entity_embedding` vs entity-specific tables
- [ ] Storage format chosen: native VECTOR vs columnar

---

## 2. Structural Checklist

For each embedding table:

- [ ] `embedding_key` — IDENTITY surrogate key present
- [ ] `{entity}_key` (or `entity_key + entity_type`) — domain FK present
- [ ] **No content columns** — no names, descriptions, text, or attributes from Domain
- [ ] `embedding_vector` — VECTOR datatype (or columnar `emb_0…emb_N` for legacy)
- [ ] `embedding_dimensions` — present and matches model output
- [ ] `embedding_model` — present and populated
- [ ] `computation_method` — ONNX / OPENAI_API / NVIDIA_NIM etc.
- [ ] `generated_dts` — temporal freshness tracking present
- [ ] `valid_from_dts` / `valid_to_dts` — embedding versioning supported
- [ ] `is_current` — BYTEINT NOT NULL DEFAULT 1, consistent with Domain module
- [ ] `PRIMARY INDEX (embedding_key)` — NUPI for temporal table
- [ ] `COMMENT ON TABLE` and `COMMENT ON COLUMN` for all objects

---

## 3. Views Checklist

- [ ] `v_{entity}_searchable` created for each embedded entity
- [ ] View joins Search to Domain — does not duplicate content
- [ ] View filters `is_current = 1` (Search) and `is_current = 1 AND is_deleted = 0` (Domain)
- [ ] `COMMENT ON VIEW` explains join strategy and intended usage
- [ ] Optional `v_{entity}_current` created if needed as ANN index input

---

## 4. No-Duplication Audit

Run this query to verify no content columns exist in Search tables:

```sql
-- Check for suspicious column names in Search database
SELECT TableName, ColumnName
FROM DBC.ColumnsV
WHERE DatabaseName = '{SearchDatabase}'
  AND ColumnName NOT IN (
    -- Expected Search module columns only:
    'embedding_key', 'entity_key', 'entity_type',
    'source_module', 'source_table', 'source_column',
    'embedded_column', 'embedding_vector', 'embedding_dimensions',
    'embedding_model', 'embedding_model_version', 'computation_method',
    'generated_dts', 'valid_from_dts', 'valid_to_dts', 'is_current',
    'generated_by', 'embedding_dims'
    -- Plus entity-specific FK columns: party_key, product_key, etc.
  )
ORDER BY TableName, ColumnName;
-- Any unexpected columns returned may indicate content duplication
```

---

## 5. Similarity Search Validation

```sql
-- Test 1: Embeddings exist and are current
SELECT entity_type, COUNT(*) AS cnt
FROM Search.entity_embedding
WHERE is_current = 1
GROUP BY entity_type;
-- Expected: row per entity type, non-zero counts

-- Test 2: Basic similarity search returns results
SELECT dt.reference_id, dt.distance
FROM TD_VECTORDISTANCE (
    ON (SELECT embedding_vector, {entity}_key
        FROM Search.{entity}_embedding
        WHERE {entity}_key = {any_valid_key} AND is_current = 1)
        AS TargetTable PARTITION BY ANY
    ON (SELECT embedding_vector, {entity}_key
        FROM Search.{entity}_embedding
        WHERE is_current = 1)
        AS ReferenceTable DIMENSION
    USING
        TargetIDColumn('{entity}_key')
        TargetFeatureColumns('embedding_vector')
        RefIDColumn('{entity}_key')
        RefFeatureColumns('embedding_vector')
        DistanceMeasure('cosine')
        TopK(5)
) AS dt
ORDER BY dt.distance;
-- Expected: 5 rows with distance values 0.0–2.0 for cosine

-- Test 3: View returns Domain content without duplication
SELECT * FROM Search.v_{entity}_searchable
WHERE {entity}_id = '{test_id}';
-- Expected: embedding columns from Search + business columns from Domain

-- Test 4: Stale embedding detection works
SELECT COUNT(*) FROM Search.entity_embedding e
INNER JOIN Domain.{Entity}_H d
    ON d.{entity}_key = e.entity_key AND d.is_current = 1
WHERE e.is_current = 1
  AND d.transaction_from_dts > e.generated_dts;
-- Expected: result (may be 0 if embeddings are fresh)
```

---

## 6. Semantic Module Registration Checklist

After building the Search module, register in Semantic:

**entity_metadata** — add one row per embedding table:
```sql
INSERT INTO Semantic.entity_metadata
(entity_name, entity_description, module_name, database_name,
 table_name, view_name, surrogate_key_column, natural_key_column,
 temporal_pattern, current_flag_column, deleted_flag_column, is_active)
VALUES
('{Entity}Embedding',
 '{Entity} vector embeddings for semantic similarity search — join to Domain for content',
 'Search', '{SearchDatabase}',
 '{entity}_embedding', 'v_{entity}_searchable',
 'embedding_key', NULL,
 'NONE', 'is_current', NULL,
 'Y');
```

**column_metadata** — register the embedding_vector column:
```sql
INSERT INTO Semantic.column_metadata
(database_name, table_name, column_name,
 business_description, is_pii, is_sensitive, data_classification,
 is_required, data_type, is_active)
VALUES
('{SearchDatabase}', '{entity}_embedding', 'embedding_vector',
 '{EmbeddingModel} {dims}-dimensional vector embedding of {entity} {source_column} for cosine similarity search',
 'N', 'N', 'INTERNAL', 'Y', 'VECTOR', 'Y');
```

**table_relationship** — register FK back to Domain:
```sql
INSERT INTO Semantic.table_relationship
(relationship_name, relationship_description,
 source_database, source_table, source_column,
 target_database, target_table, target_column,
 relationship_type, cardinality, relationship_meaning,
 is_mandatory, is_active)
VALUES
('{Entity}Embedding_To_{Entity}',
 'Search embedding references Domain entity for content retrieval via join',
 '{SearchDatabase}', '{entity}_embedding', '{entity}_key',
 '{DomainDatabase}', '{Entity}_H', '{entity}_key',
 'FOREIGN_KEY', 'M:1',
 'Each embedding references one Domain entity; join for content attributes',
 'Y', 'Y');
```

**data_product_map** — update Search module status:
```sql
UPDATE Semantic.data_product_map
SET deployment_status = 'DEPLOYED',
    deployed_dts = CURRENT_TIMESTAMP(6),
    primary_tables = '{entity}_embedding',
    primary_views = 'v_{entity}_searchable',
    updated_at = CURRENT_TIMESTAMP(6)
WHERE module_name = 'Search' AND is_active = 'Y';
```

---

## 7. Documentation Capture Checklist

- [ ] Documentation SQL written inline as `04-search/{NN}-documentation.sql` (last numbered file in module directory)
- [ ] File header includes `-- Deploy: Phase 2, after Search DDL` comment
- [ ] `dp_documentation.Module_Registry` INSERT generated (module_name = 'SEARCH', version 1.0.0)
- [ ] `dp_documentation.Design_Decision` INSERTs generated (minimum 3 decisions)
- [ ] Decision IDs follow `DD-SEARCH-{NNN}` convention
- [ ] Every decision has all ADR fields: context, alternatives_considered, rationale, consequences
- [ ] Decision categories from standard set: ARCHITECTURE | SCHEMA | NAMING | PERFORMANCE | SECURITY | INTEGRATION | OPERATIONAL
- [ ] All decisions: `decision_status = 'ACCEPTED'`, `is_current = 1`
- [ ] `dp_documentation.Change_Log` INSERT generated (CL-SEARCH-001, INITIAL_RELEASE)
- [ ] Change log includes `migration_steps` and `rollback_steps`
- [ ] `dp_documentation.Business_Glossary` INSERTs generated (minimum 3 terms)
- [ ] `dp_documentation.Query_Cookbook` INSERTs generated (minimum 1 recipe)
- [ ] All INSERTs use consistent `data_product` value
- [ ] All temporal fields: `valid_from = CURRENT_DATE`, `valid_to = DATE '9999-12-31'`
- [ ] `source_module` and `module_version` populated on every row

---

## 8. Final Pre-Delivery Checklist

- [ ] No content columns in any Search table (audit query run and passed)
- [ ] All embedding tables have `COMMENT ON TABLE` + `COMMENT ON COLUMN` for every column
- [ ] `v_{entity}_searchable` view exists and tested for each embedded entity
- [ ] TD_VectorDistance test query returns results with valid distance scores
- [ ] Embedding refresh strategy documented and pipeline exists or planned
- [ ] ANN index decision documented with rationale
- [ ] Semantic module updated: entity_metadata, column_metadata, table_relationship, data_product_map
- [ ] Teradata version compatibility confirmed for VECTOR datatype usage
- [ ] Documentation capture complete — all dp_documentation INSERTs generated
