---
name: ai-native-semantic-design
description: >
  Design the Semantic module for an AI-Native Data Product on Teradata.
  Produces DDL for metadata tables (entity_metadata, column_metadata, table_relationship,
  naming_standard, data_product_map), the recursive multi-hop path discovery view
  (v_relationship_paths), standard discovery views, and seed INSERT statements for
  any business domain. Use this skill whenever the user asks to design, build, or
  populate a Semantic module, define table relationships for agent SQL generation,
  create a knowledge/metadata layer, or enable multi-hop join path discovery in
  an AI-Native data product.
---

# AI-Native Semantic Module Design Skill

## Terminology (Critical)

| Term | Means |
|------|-------|
| **Entity** | A table (e.g., `Party_H` is an entity) |
| **Attribute** | A column (e.g., `party_key` is an attribute) |
| **Relationship** | How two tables join |

The Semantic module stores **schema metadata** (hundreds of rows) — never instance data (millions of rows).

---

## Role in the Architecture

The Semantic module is built **second**, immediately after Domain. It describes **all six modules** — including itself — giving agents a single place to discover structure and join paths without human instruction.

```
data_product_map     → "What modules exist and where are they?"
entity_metadata      → "What tables exist in each module?"
column_metadata      → "What do the columns mean?"
table_relationship   → "How do tables join?"
naming_standard      → "What do naming conventions mean?"
v_relationship_paths → "How do I join table A to table B (multi-hop)?"
```

---

## Design Workflow

### Step 1 — Confirm Inputs
Gather from context or ask:
- Data product name (e.g., Customer360)
- Database layout: **Separate DBs** (`Customer360_Domain`, `Customer360_Semantic`, …) or **Single DB** (`Customer360` with `D_`, `S_` prefixes)?
- Which modules are deployed or planned?
- All Domain entities and their relationships (from the Domain module design)
- Enterprise naming conventions to document in `naming_standard`

### Step 2 — Choose Optional Tables

| Table | Include When |
|-------|-------------|
| `ontology` | Formal taxonomies (FIBO concepts, product hierarchies) needed |
| `business_rule` | Data validation rules should be queryable by agents |
| `data_contract_catalog` | Formal ODCS/data contracts in use |

The four **core tables + v_relationship_paths** are always required.

### Step 3 — Build in This Order

1. **`data_product_map`** — register all modules first (agents start here)
2. **`entity_metadata`** — one row per table across all modules
3. **`naming_standard`** — document all suffixes, prefixes, patterns
4. **`column_metadata`** — focus on PII/sensitive/classification flags; business meanings supplement `COMMENT ON COLUMN` in Domain DDL
5. **`table_relationship`** — one row per directional join; be exhaustive
6. **`v_relationship_paths`** — create view last (depends on `table_relationship`)
7. **`v_entity_catalog`** and **`v_entity_schema`** — agent convenience views

### Step 4 — Populate Data
Seed data is as important as the DDL. For each step above, produce `INSERT` statements alongside the `CREATE TABLE`. See `references/population-patterns.md` for templates.

### Step 5 — Documentation Capture

Read `../ai-native-documentation-design/references/documentation-capture.md` for the full protocol, SQL templates, and ID conventions.

**Module short name:** `SEMANTIC` — Decision IDs: `DD-SEMANTIC-{NNN}`, Change log: `CL-SEMANTIC-{NNN}`, Cookbook: `QC-SEMANTIC-{NNN}`

**Typical decisions to capture:**

| Decision | Category | Affects |
|----------|----------|---------|
| Relationship modeling approach (forward-only registration) | ARCHITECTURE | `table_relationship` |
| Metadata standards (entity_metadata column set) | SCHEMA | All Semantic tables |
| Optional tables included/excluded (ontology, business_rule, etc.) | ARCHITECTURE | Semantic module scope |
| Hop limit strategy for v_relationship_paths | PERFORMANCE | `v_relationship_paths` |
| PII classification scheme | SECURITY | `column_metadata` |
| Database layout pattern (SEPARATE_DB vs SINGLE_DB_PREFIX) | NAMING | `data_product_map` |

**Typical glossary terms:** entity metadata, column metadata, table relationship, naming standard, data product map, relationship path, hop count

**Typical cookbook entries:** multi-hop join path discovery query, entity catalog query, PII column discovery

**Required outputs:**
1. `dp_documentation.Module_Registry` INSERT (1 — module registration)
2. `dp_documentation.Design_Decision` INSERTs (minimum 3 — key architectural/schema choices)
3. `dp_documentation.Change_Log` INSERT (1 — CL-SEMANTIC-001, INITIAL_RELEASE v1.0.0)
4. `dp_documentation.Business_Glossary` INSERTs (minimum 3 — domain terms introduced)
5. `dp_documentation.Query_Cookbook` INSERTs (minimum 1 — key query patterns)

### Step 6 — Validate
Run the agent discovery test in `references/checklists.md`.

---

## Key Design Rules

**`table_relationship`**
- Register every FK join, hierarchy, and associative relationship
- Both directions are handled by `v_relationship_paths` — only register the forward direction (source = table with FK column)
- `relationship_type` values: `FOREIGN_KEY`, `HIERARCHY`, `ASSOCIATIVE`
- `cardinality` values: `1:1`, `1:M`, `M:1`, `M:M`

**`v_relationship_paths`**
- This is the critical agent capability — enables SQL generation without hardcoding joins
- Bidirectional traversal up to 5 hops, loop detection included
- **Do not modify the CTE structure** — use the validated template in `references/semantic-views.md`

**`data_product_map`**
- `naming_pattern`: `SEPARATE_DB` or `SINGLE_DB_PREFIX`
- `deployment_status`: `DEPLOYED`, `PLANNED`, `DEPRECATED`
- One row per module; include Semantic itself

**Agent first-query pattern:**
```sql
-- Agent entry point: discover all modules
SELECT module_name, database_name, primary_tables, deployment_status
FROM Semantic.data_product_map
WHERE is_active = 'Y'
ORDER BY module_name;
```

---

## Quick Reference: Expected Row Counts

| Table | Typical Range |
|-------|--------------|
| `data_product_map` | 6 rows (one per module) |
| `entity_metadata` | 10–50 rows |
| `naming_standard` | 10–30 rows |
| `column_metadata` | 100–500 rows |
| `table_relationship` | 20–100 rows |

---

## Reference Files

| File | Read When |
|------|-----------|
| `references/semantic-tables.md` | Writing DDL for the 5 core tables |
| `references/semantic-views.md` | Building v_relationship_paths and discovery views |
| `references/population-patterns.md` | Writing seed INSERT statements for any domain |
| `references/checklists.md` | Final review before delivering design |
| `../ai-native-documentation-design/references/documentation-capture.md` | Documentation Capture step — SQL templates and ID conventions |
