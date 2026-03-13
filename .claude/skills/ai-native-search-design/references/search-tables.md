# Search Tables & Views Reference
## DDL Templates for AI-Native Search Module

Replace `Search` with your actual database name (e.g., `Customer360_Search`).
Replace `Domain` with your Domain database name (e.g., `Customer360_Domain`).

---

## 1. Generic Embedding Table (Multi-Entity Pattern)

Use when managing embeddings for multiple entity types in one table.

```sql
CREATE TABLE Search.entity_embedding (
    embedding_key           INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for embedding record',

    -- Entity reference (KEY ONLY — no content duplication)
    entity_key              BIGINT NOT NULL
        COMMENT 'FK to Domain entity — references party_key, product_key, etc. NO content stored here',
    entity_type             VARCHAR(50) NOT NULL
        COMMENT 'Domain entity type: PARTY, PRODUCT, DOCUMENT — identifies which Domain table to join',

    -- Source identification
    source_module           VARCHAR(50)
        COMMENT 'Module where source entity resides: Domain, Prediction',
    source_table            VARCHAR(100)
        COMMENT 'Source table name: Party_H, Product_H, Document_H',
    source_column           VARCHAR(100)
        COMMENT 'Column that was embedded: description, notes, combined_text',

    -- Vector (Teradata native — requires v20.00.26.XX+)
    embedding_vector        VECTOR
        COMMENT 'Vector embedding stored as Teradata native VECTOR (FLOAT32) datatype for similarity search',
    embedding_dimensions    INTEGER NOT NULL
        COMMENT 'Number of embedding dimensions: 384, 768, 1536 — must match embedding_model output',

    -- Embedding provenance
    embedding_model         VARCHAR(100) NOT NULL
        COMMENT 'Model that generated the embedding: bge-small-en-v1.5, text-embedding-ada-002',
    embedding_model_version VARCHAR(20)
        COMMENT 'Model version for reproducibility tracking',
    computation_method      VARCHAR(50)
        COMMENT 'Generation method: ONNX (in-database), OPENAI_API, NVIDIA_NIM, HUGGINGFACE_API',

    -- Temporal tracking
    generated_dts           TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When this embedding was generated — tracks freshness of semantic representation',
    valid_from_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When this embedding version became valid',
    valid_to_dts            TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'When superseded — default 9999 = current embedding',
    is_current              BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Current embedding flag: 1 = current, 0 = historical (replaced by model update or content change)',

    -- Audit
    generated_by            VARCHAR(100)
        COMMENT 'User or process that triggered embedding generation: ML pipeline, batch job, API'
)
PRIMARY INDEX (embedding_key);   -- NUPI: allows multiple versions per entity

COMMENT ON TABLE Search.entity_embedding IS
'Vector embeddings for Domain entities — stores vectors and keys only.
 NO content duplication: join to Domain table via entity_key + entity_type for attributes.
 Current embeddings: is_current = 1. Filter by entity_type for entity-specific searches.';
```

---

## 2. Entity-Specific Embedding Table

Use for high-volume single-entity workloads where a typed FK and dedicated index improve performance.
Follow this pattern for each entity that warrants its own table.

```sql
CREATE TABLE Search.{entity}_embedding (
    embedding_key           INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for embedding record',

    -- Typed FK to Domain (cleaner than entity_key + entity_type)
    {entity}_key            BIGINT NOT NULL
        COMMENT 'FK to Domain.{Entity}_H.{entity}_key — NO content stored here',

    -- Source identification
    embedded_column         VARCHAR(100)
        COMMENT 'Column embedded from {Entity}_H: description, notes, combined_text',

    -- Vector
    embedding_vector        VECTOR
        COMMENT 'FLOAT32 vector embedding for {entity} semantic similarity search',
    embedding_dimensions    INTEGER NOT NULL DEFAULT 384
        COMMENT 'Embedding dimensions — must match embedding_model output',
    embedding_model         VARCHAR(100) NOT NULL
        COMMENT 'Model used: bge-small-en-v1.5, text-embedding-ada-002, etc.',
    computation_method      VARCHAR(50)
        COMMENT 'ONNX | OPENAI_API | NVIDIA_NIM | HUGGINGFACE_API',

    -- Temporal
    generated_dts           TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When embedding was generated',
    valid_from_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'Embedding validity start',
    valid_to_dts            TIMESTAMP(6) WITH TIME ZONE NOT NULL
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00'
        COMMENT 'Embedding validity end — 9999 = current',
    is_current              BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Current embedding: 1 = current, 0 = historical'
)
PRIMARY INDEX ({entity}_key);   -- NUPI on domain FK for join co-location

COMMENT ON TABLE Search.{entity}_embedding IS
'{Entity} vector embeddings — stores {entity}_key and embedding_vector only.
 Join to Domain.{Entity}_H on {entity}_key for all content attributes.
 Current: is_current = 1.';
```

---

## 3. Columnar Storage (Legacy / Pre-20.00.26.XX)

Use when the Teradata version does not support the native VECTOR datatype, or when per-dimension analytics are needed.

```sql
CREATE TABLE Search.{entity}_embedding_columnar (
    embedding_key       INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for embedding record',
    {entity}_key        BIGINT NOT NULL
        COMMENT 'FK to Domain.{Entity}_H.{entity}_key',
    embedded_column     VARCHAR(100)
        COMMENT 'Source column embedded',

    -- Embedding as individual FLOAT columns (one per dimension)
    -- For 384-dim model: emb_0 through emb_383
    emb_0   FLOAT, emb_1   FLOAT, emb_2   FLOAT,
    -- ... continue to emb_{N-1} where N = embedding_dimensions
    -- TD_VectorDistance uses range notation: '[emb_0:emb_383]'

    embedding_model     VARCHAR(100) NOT NULL
        COMMENT 'Model used to generate embedding',
    embedding_dims      INTEGER NOT NULL
        COMMENT 'Total dimensions — determines emb_0 to emb_{N-1} range',
    generated_dts       TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When embedding was generated',
    is_current          BYTEINT NOT NULL DEFAULT 1
        COMMENT '1 = current, 0 = historical'
)
PRIMARY INDEX ({entity}_key);

COMMENT ON TABLE Search.{entity}_embedding_columnar IS
'{Entity} embeddings in columnar format for Teradata < 20.00.26.XX.
 Reference dimensions as ''[emb_0:emb_{N-1}]'' in TD_VectorDistance.
 Migrate to native VECTOR format when upgrading to v20.00.26.XX+.';
```

---

## 4. Standard Views

### `v_{entity}_searchable` — Search + Domain Content (REQUIRED)

The primary agent access surface. Joins Search and Domain so agents get full context in one query.

```sql
CREATE VIEW Search.v_{entity}_searchable AS
SELECT
    -- Entity identification (from Domain)
    {dom}.{entity}_key,
    {dom}.{entity}_id,

    -- Business content (from Domain — NOT duplicated in Search)
    -- [Include columns agents commonly need for search results]
    {dom}.{content_column},     -- e.g., legal_name, description, notes
    -- [Add other relevant Domain columns]

    -- Embedding metadata (from Search)
    emb.embedding_vector,
    emb.embedding_model,
    emb.embedding_dimensions,
    emb.embedded_column,
    emb.generated_dts

FROM Domain.{Entity}_H {dom}
INNER JOIN Search.{entity}_embedding emb
    ON emb.{entity}_key = {dom}.{entity}_key
   AND emb.is_current = 1
WHERE {dom}.is_current = 1
  AND {dom}.is_deleted = 0;

COMMENT ON VIEW Search.v_{entity}_searchable IS
'{Entity} embeddings with Domain content — joins Search vectors with Domain attributes.
 Use as the input surface for TD_VectorDistance similarity queries.
 Provides entity context without duplicating content in Search module.';
```

### `v_{entity}_current` — Current Embeddings Only (Optional)

Lightweight view when Domain content is not needed (e.g., to feed an ANN index).

```sql
CREATE VIEW Search.v_{entity}_current AS
SELECT
    embedding_key,
    {entity}_key,
    embedded_column,
    embedding_vector,
    embedding_dimensions,
    embedding_model,
    generated_dts
FROM Search.{entity}_embedding
WHERE is_current = 1;

COMMENT ON VIEW Search.v_{entity}_current IS
'Current {entity} embeddings only — no Domain join.
 Use as ReferenceTable input to TD_VectorDistance for similarity search.';
```

---

## 5. Anti-Pattern Reference (What NOT to Do)

```sql
-- ❌ WRONG: Content duplication in Search table
CREATE TABLE Search.product_embedding_BAD (
    embedding_key    INTEGER,
    product_key      BIGINT,
    embedding_vector VECTOR,
    product_name     VARCHAR(200),       -- ❌ already in Domain.Product_H
    product_description VARCHAR(2000),   -- ❌ already in Domain.Product_H
    category         VARCHAR(100),       -- ❌ already in Domain.Product_H
    price            DECIMAL(10,2)       -- ❌ already in Domain.Product_H
);
-- Result: wasted storage + sync drift when Domain data updates

-- ✅ CORRECT: Keys only, join for content
CREATE TABLE Search.product_embedding_GOOD (
    embedding_key    INTEGER,
    product_key      BIGINT,             -- Key to Domain only
    embedded_column  VARCHAR(100),       -- 'product_description' (metadata, not content)
    embedding_vector VECTOR,
    embedding_model  VARCHAR(100),
    is_current       BYTEINT
);
-- Content accessed at query time via JOIN to Domain.Product_H
```
