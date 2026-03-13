# Search Query Patterns Reference
## TD_VectorDistance, RAG, ANN Indexing, Discovery

---

## 1. TD_VectorDistance — Core Similarity Search

TD_VectorDistance executes in parallel across all Teradata AMPs. Always use it for similarity search — never compute distance in application code.

### Basic Pattern: Find Top-K Similar Entities

```sql
-- Find top 10 entities similar to a known entity
SELECT
    ref.entity_key          AS similar_entity_key,
    dt.distance             AS similarity_distance
FROM TD_VECTORDISTANCE (
    -- TargetTable: the query vector (what you're searching for)
    ON (
        SELECT embedding_vector, entity_key
        FROM Search.entity_embedding
        WHERE entity_key = {query_entity_key}
          AND entity_type = '{ENTITY_TYPE}'
          AND is_current = 1
    ) AS TargetTable PARTITION BY ANY

    -- ReferenceTable: the search corpus
    ON (
        SELECT embedding_vector, entity_key
        FROM Search.entity_embedding
        WHERE entity_type = '{ENTITY_TYPE}'
          AND is_current = 1
    ) AS ReferenceTable DIMENSION

    USING
        TargetIDColumn('entity_key')
        TargetFeatureColumns('embedding_vector')
        RefIDColumn('entity_key')
        RefFeatureColumns('embedding_vector')
        DistanceMeasure('cosine')       -- cosine | euclidean | manhattan
        TopK(10)
) AS dt
INNER JOIN Search.entity_embedding ref
    ON ref.entity_key = dt.reference_id
   AND ref.is_current = 1
ORDER BY dt.distance;
```

### With Domain Content (Standard Pattern)

Search provides similarity ranking; Domain provides content. Never the other way around.

```sql
-- Find similar {entity} records with full Domain attributes
SELECT
    {dom}.{entity}_id,
    {dom}.{content_column},
    -- [other Domain columns needed in results]
    dt.distance             AS similarity_score
FROM TD_VECTORDISTANCE (
    ON {query_cte} AS TargetTable PARTITION BY ANY
    ON Search.v_{entity}_current AS ReferenceTable DIMENSION
    USING
        TargetIDColumn('{entity}_key')
        TargetFeatureColumns('embedding_vector')
        RefIDColumn('{entity}_key')
        RefFeatureColumns('embedding_vector')
        DistanceMeasure('cosine')
        TopK(10)
) AS dt
-- Join to Domain for content (NOT stored in Search)
INNER JOIN Domain.{Entity}_H {dom}
    ON {dom}.{entity}_key = dt.reference_id
   AND {dom}.is_current = 1
   AND {dom}.is_deleted = 0
ORDER BY dt.distance;
```

### Columnar Format (Range Notation)

For tables using the columnar pattern (`emb_0 … emb_383`):

```sql
USING
    TargetFeatureColumns('[emb_0:emb_383]')   -- range covers all dimensions
    RefFeatureColumns('[emb_0:emb_383]')
    DistanceMeasure('cosine')
    TopK(10)
```

---

## 2. RAG Pattern — Retrieval Augmented Generation

Retrieve the most semantically relevant records to inject as context into an LLM prompt.

```sql
-- Retrieve top-5 most relevant documents for a query
-- Assumes query_embedding CTE provides the vectorised user query

WITH query_embedding AS (
    -- Option A: Embedding already stored (entity-to-entity similarity)
    SELECT embedding_vector
    FROM Search.entity_embedding
    WHERE entity_key = {seed_entity_key} AND is_current = 1

    -- Option B: In-database generation via ONNXEmbeddings
    -- SELECT embedding FROM TD_MLPREDICT(...) where input = 'user query text'
)
SELECT
    {dom}.{document_id_column},
    {dom}.{title_column},
    {dom}.{content_column},        -- Full content from Domain (not duplicated)
    dt.distance                    AS relevance_score
FROM TD_VECTORDISTANCE (
    ON query_embedding AS TargetTable PARTITION BY ANY
    ON (
        SELECT embedding_vector, entity_key
        FROM Search.entity_embedding
        WHERE entity_type = '{DOCUMENT_TYPE}'
          AND is_current = 1
    ) AS ReferenceTable DIMENSION
    USING
        RefIDColumn('entity_key')
        RefFeatureColumns('embedding_vector')
        DistanceMeasure('cosine')
        TopK(5)
) AS dt
INNER JOIN Domain.{Document_H} {dom}
    ON {dom}.{entity}_key = dt.reference_id
   AND {dom}.is_current = 1
   AND {dom}.is_deleted = 0
ORDER BY dt.distance
QUALIFY ROW_NUMBER() OVER (ORDER BY dt.distance) <= 5;
```

**LLM integration**: Pass the `{content_column}` values from the result set as context chunks in the prompt. Order by `relevance_score` ascending (lower distance = more similar).

---

## 3. ANN Indexing — Enterprise Vector Store API

Use the Teradata Enterprise Vector Store Python API for large-scale (100K+) vector datasets. Direct SQL DDL is not used for index creation.

### KMEANS (IVF-style) — Batch / Offline Search

Best for large static datasets where approximate results are acceptable and the index can be periodically rebuilt.

```python
from teradatagenai import VectorStore

vs = VectorStore(
    conn=td_connection,
    database='{SearchDatabase}',
    table='{entity}_embedding'
)

# Build KMEANS index
vs.create(
    search_algorithm='KMEANS',
    vector_column='embedding_vector',
    id_column='{entity}_key',
    train_numcluster=4,         # Tune: sqrt(total_vectors / 1000) is a starting heuristic
    max_iternum=50,             # Convergence iterations
    stop_threshold=0.0395,      # Convergence precision
    metric='COSINE'
)

# Query with KMEANS
results = vs.search(
    query_vector=query_emb,
    top_k=10,
    nprobe=2                    # Number of clusters to search (higher = more accurate, slower)
)
```

### HNSW (Graph-based) — Real-Time Interactive Search

Best for interactive search where latency matters and the vector set is frequently updated.

```python
# Build HNSW index
vs.create(
    search_algorithm='hnsw',
    vector_column='embedding_vector',
    id_column='{entity}_key',
    ef_search=50,               # Accuracy-speed tradeoff: 50–200 typical range
    metric='COSINE'
)

# Query with HNSW
results = vs.search(
    query_vector=query_emb,
    top_k=10
)
```

### Index Strategy Decision

| Condition | Index |
|-----------|-------|
| < 50K vectors | None — use TD_VectorDistance exact search |
| 100K+ vectors, batch queries | KMEANS |
| 100K+ vectors, real-time queries | HNSW |
| Vectors updated frequently | HNSW (rebuilds incrementally) |
| Vectors mostly static, rebuilt periodically | KMEANS (cheaper to rebuild) |

---

## 4. Embedding Generation Patterns

### Option A: In-Database via ONNXEmbeddings (Preferred for Teradata)

```sql
-- Generate embedding for new entity using stored ONNX model
-- (Model must be loaded into Teradata's model catalog)
SELECT TD_MLPREDICT(
    USING
        ModelName('bge-small-en-v1.5')
        InputColumns('description')   -- column to embed
) FROM Domain.{Entity}_H
WHERE {entity}_key = {new_key}
  AND is_current = 1;
```

### Option B: External API → Teradata INSERT

```python
# Generate via OpenAI/HuggingFace API, insert into Search table
import openai

response = openai.embeddings.create(
    model='text-embedding-ada-002',
    input=entity_text
)
embedding = response.data[0].embedding

# Insert into Search table
cursor.execute("""
    INSERT INTO Search.{entity}_embedding
    ({entity}_key, embedded_column, embedding_vector,
     embedding_dimensions, embedding_model, computation_method,
     generated_dts, valid_from_dts, is_current)
    VALUES (?, ?, ?::VECTOR, ?, ?, ?, CURRENT_TIMESTAMP(6), CURRENT_TIMESTAMP(6), 1)
""", [entity_key, 'description', str(embedding), 1536, 'text-embedding-ada-002', 'OPENAI_API'])
```

### Embedding Refresh Pattern

When Domain entity content changes, the embedding must be regenerated:

```sql
-- Step 1: Expire current embedding
UPDATE Search.{entity}_embedding
SET valid_to_dts = CURRENT_TIMESTAMP(6),
    is_current = 0
WHERE {entity}_key = {changed_key}
  AND is_current = 1;

-- Step 2: Insert new embedding (from regeneration process)
INSERT INTO Search.{entity}_embedding
({entity}_key, embedded_column, embedding_vector,
 embedding_dimensions, embedding_model, computation_method,
 generated_dts, valid_from_dts, valid_to_dts, is_current)
VALUES
({changed_key}, 'description', {new_vector}::VECTOR,
 384, 'bge-small-en-v1.5', 'ONNX',
 CURRENT_TIMESTAMP(6), CURRENT_TIMESTAMP(6),
 TIMESTAMP '9999-12-31 23:59:59.999999+00:00', 1);
```

---

## 5. Agent Discovery Queries

Agents use these to understand what is searchable without prior knowledge.

```sql
-- What entity types have embeddings?
SELECT DISTINCT entity_type, source_table,
    COUNT(*) AS embedding_cnt,
    MAX(generated_dts) AS latest_generation
FROM Search.entity_embedding
WHERE is_current = 1
GROUP BY entity_type, source_table
ORDER BY entity_type;

-- What embedding models are in use?
SELECT DISTINCT
    embedding_model,
    embedding_dimensions,
    COUNT(*) AS vector_cnt
FROM Search.entity_embedding
WHERE is_current = 1
GROUP BY embedding_model, embedding_dimensions
ORDER BY embedding_model;

-- Are embeddings fresh relative to Domain updates?
SELECT
    e.entity_type,
    COUNT(*) AS stale_cnt
FROM Search.entity_embedding e
INNER JOIN Domain.{Entity}_H d
    ON d.{entity}_key = e.entity_key
   AND d.is_current = 1
WHERE e.is_current = 1
  AND d.transaction_from_dts > e.generated_dts  -- Domain updated after embedding generated
GROUP BY e.entity_type;
```
