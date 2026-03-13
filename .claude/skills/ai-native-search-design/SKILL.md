---
name: ai-native-search-design
description: >
  Design the Search module for an AI-Native Data Product on Teradata.
  Produces DDL for vector embedding tables, standard views combining Search and Domain content,
  similarity search queries using TD_VectorDistance, RAG retrieval patterns, and ANN indexing
  configuration (KMEANS, HNSW). Use this skill whenever the user asks to design or build a Search
  module, add semantic search or similarity search to a data product, implement RAG (Retrieval
  Augmented Generation) patterns on Teradata, work with vector embeddings or the VECTOR datatype,
  or configure KMEANS or HNSW vector indexes.
---

# AI-Native Search Module Design Skill

## Core Principle (Never Violate)

**Store vectors + keys only. Never duplicate content from Domain.**

```
Search table:          Domain table:
embedding_key          party_key
entity_key  ──────────> party_id
entity_type            legal_name
embedding_vector       description   ← content lives here only
embedding_model        ...all attributes
```

All content is retrieved by joining Search to Domain at query time. Teradata co-location makes this efficient. Duplicating content wastes storage and creates sync risk.

---

## Role in the Architecture

Search is built in **Phase 2** (after Domain and Semantic). It embeds Domain entities to enable:

| Use Case | Pattern |
|----------|---------|
| Semantic search | "Find products similar to X" |
| RAG | Retrieve relevant documents for LLM context |
| Similarity analysis | Cluster/group entities by meaning |
| Content discovery | Agents find relevant data without exact keywords |
| Multi-modal search | Text, images, structured data embeddings |

**Dependencies**: Domain module must exist first. Semantic module should describe embedding strategies in `column_metadata`.

---

## Design Workflow

### Step 1 — Gather Requirements

Confirm or ask for:
- Which Domain entities need embeddings (Party, Product, Document, …)
- Which column(s) to embed per entity (description, notes, combined_text, …)
- Embedding model and output dimensions
- Expected dataset size (drives indexing strategy)
- Search latency requirement (real-time vs batch)
- Teradata version (VECTOR datatype requires 20.00.26.XX+)

### Step 2 — Key Decisions

**Storage format:**

| Condition | Choice |
|-----------|--------|
| Teradata ≥ 20.00.26.XX | Native `VECTOR` datatype (advocated) |
| Teradata 17.20–19.x | Columnar (`emb_0 … emb_N FLOAT`) |
| Need per-dimension analytics | Columnar |

**Table pattern:**

| Condition | Choice |
|-----------|--------|
| Multiple entity types, unified management | Single generic `entity_embedding` table (use `entity_type` column) |
| One entity type, high volume, maximum query performance | Entity-specific table (e.g., `party_embedding`) |

Both patterns store keys only. Entity-specific tables allow a more specific FK (`party_key` instead of `entity_key + entity_type`) and a cleaner primary index.

**Indexing strategy:**

| Dataset Size | Latency Need | Strategy |
|-------------|-------------|----------|
| < 50K vectors | Any | Exact search (TD_VectorDistance, no index) |
| 100K+ vectors | Batch/offline | KMEANS (IVF-style clustering) |
| 100K+ vectors | Real-time interactive | HNSW (graph-based) |

**Distance metric:**

| Data Type | Metric |
|-----------|--------|
| Text/semantic embeddings | Cosine (default) |
| Spatial / geographic | Euclidean |
| High-dimensional sparse | Manhattan |

### Step 3 — Build in This Order

1. Embedding table(s) — DDL with full `COMMENT ON` statements
2. Standard views — `v_{entity}_searchable` (Search + Domain join) and optionally `v_{entity}_current`
3. Similarity search queries — parameterised TD_VectorDistance patterns
4. ANN index configuration — KMEANS or HNSW if needed
5. Register in Semantic module — add rows to `entity_metadata`, `column_metadata`, `table_relationship`

### Step 4 — Documentation Capture

Read `../ai-native-documentation-design/references/documentation-capture.md` for the full protocol, SQL templates, and ID conventions.

**Module short name:** `SEARCH` — Decision IDs: `DD-SEARCH-{NNN}`, Change log: `CL-SEARCH-{NNN}`, Cookbook: `QC-SEARCH-{NNN}`

**Typical decisions to capture:**

| Decision | Category | Affects |
|----------|----------|---------|
| Vector storage format (native VECTOR vs columnar) | ARCHITECTURE | Embedding tables |
| Table pattern (generic entity_embedding vs entity-specific) | SCHEMA | Embedding tables |
| Indexing strategy (exact / KMEANS / HNSW) | PERFORMANCE | Embedding tables |
| Distance metric (cosine / euclidean / manhattan) | PERFORMANCE | Similarity queries |
| Embedding model selection and dimensions | ARCHITECTURE | All embeddings |
| Update strategy (on insert, daily batch, on-demand) | OPERATIONAL | Embedding pipeline |

**Typical glossary terms:** vector embedding, similarity search, cosine distance, ANN index, RAG retrieval, embedding dimensions, searchable view

**Typical cookbook entries:** similarity search query (TD_VectorDistance), RAG retrieval pattern, stale embedding detection

**Required outputs:**
1. `dp_documentation.Module_Registry` INSERT (1 — module registration)
2. `dp_documentation.Design_Decision` INSERTs (minimum 3 — key architectural/schema choices)
3. `dp_documentation.Change_Log` INSERT (1 — CL-SEARCH-001, INITIAL_RELEASE v1.0.0)
4. `dp_documentation.Business_Glossary` INSERTs (minimum 3 — domain terms introduced)
5. `dp_documentation.Query_Cookbook` INSERTs (minimum 1 — key query patterns)

### Step 5 — Validate

Run the discovery and search test in `references/checklists.md`.

---

## Quick Reference

**Embedding table essentials:**
```
embedding_key    INTEGER GENERATED ALWAYS AS IDENTITY
entity_key       BIGINT NOT NULL         -- FK to Domain (NO content here)
entity_type      VARCHAR(50)             -- 'PARTY', 'PRODUCT', etc.
source_column    VARCHAR(100)            -- What was embedded: 'description'
embedding_vector VECTOR                  -- FLOAT32(n) native type
embedding_dimensions INTEGER            -- 384, 768, 1536
embedding_model  VARCHAR(100)           -- 'bge-small-en-v1.5'
generated_dts    TIMESTAMP(6) WITH TIME ZONE
is_current       BYTEINT NOT NULL DEFAULT 1  -- 1/0, consistent with Domain module
PRIMARY INDEX (embedding_key)           -- NUPI (temporal, multiple versions allowed)
```

**Standard filter for current embeddings:**
```sql
WHERE is_current = 1     -- BYTEINT, consistent with Domain module
```

**Standard view naming:**
```
v_{entity}_searchable   -- Embedding + Domain content joined (agent query surface)
v_{entity}_current      -- Current embeddings only (no Domain join)
```

**Common embedding models:**

| Model | Dims | Best For |
|-------|------|----------|
| bge-small-en-v1.5 | 384 | Efficient text search (Teradata native via ONNX) |
| bge-base-en-v1.5 | 768 | Higher accuracy text search |
| all-MiniLM-L6-v2 | 384 | Fast sentence embeddings |
| text-embedding-ada-002 | 1536 | OpenAI API (external) |
| CLIP | 512 | Multi-modal text + images |

**Teradata prerequisites:**
- VECTOR datatype: v20.00.26.XX+
- TD_VectorDistance: ClearScape Analytics
- ONNXEmbeddings: ClearScape Analytics
- Python APIs: `teradatagenai`, `langchain-teradata`, `teradataml`

---

## Semantic Module Registration

After building the Search module, register it in Semantic:

```sql
-- entity_metadata: register each embedding table
-- column_metadata: document embedding_vector with model name and dims
-- table_relationship: party_embedding.party_key → Domain.Party_H.party_key
-- data_product_map: update Search module row from PLANNED to DEPLOYED
```

See `references/checklists.md` for the full registration checklist.

---

## Reference Files

| File | Read When |
|------|-----------|
| `references/search-tables.md` | Writing DDL for embedding tables and standard views |
| `references/search-queries.md` | Writing TD_VectorDistance queries, RAG patterns, ANN index config |
| `references/checklists.md` | Final review, Semantic module registration, agent validation |
| `../ai-native-documentation-design/references/documentation-capture.md` | Documentation Capture step — SQL templates and ID conventions |
