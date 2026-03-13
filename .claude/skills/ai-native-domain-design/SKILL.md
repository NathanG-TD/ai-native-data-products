---
name: ai-native-domain-design
description: >
  Design the Domain/Subject Data module for an AI-Native Data Product on Teradata.
  Produces DDL templates, naming conventions, view definitions, cross-module integration patterns,
  and agent-discoverability checklist for any business domain (Finance, Healthcare, Retail, etc.).
  Use this skill whenever the user asks to design, review, or validate a Domain module,
  define entity structures, apply AI-Native data product standards, or produce Teradata DDL
  for master data / reference data / relationship tables as part of an AI-Native architecture.
---

# AI-Native Domain Module Design Skill

## Architecture Context (Read First)

An AI-Native Data Product has **6 modules**. The Domain module is always built first — everything else joins back to it.

| Module | Prefix | Purpose |
|--------|--------|---------|
| **Domain** (this skill) | `D_` | Authoritative business entities, relationships, reference data |
| Semantic | `S_` | Ontologies, query patterns, naming standards |
| Prediction | `P_` | ML features, model outputs |
| Search | `E_` | Vector embeddings, similarity search |
| Observability | `O_` | Events, quality metrics, audit |
| Memory | `M_` | Agent session state |

**Database layout**: Single DB with module-prefixed tables (small deployments) OR separate DB per module e.g. `CustomerDomain`, `CustomerSemantic` (enterprise default).

---

## Design Workflow

### Step 1 — Gather Requirements
Ask the user for (or infer from context):
- Business domain name (e.g. Customer, Product, Agreement)
- Core entities and their relationships
- Source system(s) and natural key formats
- Temporal requirements (do they need point-in-time for ML?)
- Deployment scale (single team vs enterprise)
- Any existing enterprise naming standards to follow

### Step 2 — Choose Key Decisions
Use the decision tables below; document choices before writing DDL.

**Temporal Strategy**
| Condition | Choice |
|-----------|--------|
| Point-in-time ML features needed, OR late-arriving facts, OR full audit | **Bi-temporal** (advocated) |
| Simple history only, no corrections | Type 2 SCD acceptable |
| No history needed | Current-only (no `_H` suffix) |

**Column Set** (use for Domain `_H` tables)
| Table Volume | Recommended Set |
|-------------|----------------|
| < 10M rows | Core + Extended optional |
| 10M–100M rows | Core + FK refs to Observability |
| > 100M rows | Core only; join Observability by entity_key |

**Key Strategy**: Surrogate key + Natural key (both required). See `references/implementation-guidance.md` for sequence strategy.

**Delete Strategy**: Soft delete (`is_deleted`) is the advocated default. Hard delete only when storage is extremely constrained AND no audit/ML dependency.

### Step 3 — Design Entities
For each entity, produce:
1. `{Entity}_H` history table (DDL template in `references/domain-structure.md`)
2. `{Entity}_Current` view (required)
3. `{Entity}_Enriched` view (optional, for common join patterns)
4. Reference data tables `{Reference}_R` where needed
5. Relationship tables `{Entity1}{Entity2}_H` where needed

**AI-Native requirements for every table/view:**
- `COMMENT ON TABLE` — business purpose, not technical description
- `COMMENT ON COLUMN` for every column — meaning, context, constraints
- Consistent patterns across ALL entities (agents learn by pattern, not by exception)
- Descriptive foreign keys: `party_key`, not `fk1`

### Step 4 — Cross-Module Integration
Decide FK pattern for other modules referencing Domain:
- **Pattern A** (flexible): `entity_key BIGINT + entity_type VARCHAR(50)` — use when module references many entity types
- **Pattern B** (type-safe): explicit `party_key`, `product_key` columns — use when module references few known types

All modules JOIN back to Domain; they never duplicate Domain attributes.

### Step 5 — Physical Design
- **Primary Index**: Choose based on dominant access pattern (surrogate key typical for NUPI on temporal tables)
- **Temporal tables**: Always use `PRIMARY INDEX` (NUPI), never `UNIQUE PRIMARY INDEX` — multiple versions share same key
- **Reference tables**: `UNIQUE PRIMARY INDEX ({code}, effective_date)` is appropriate
- **Secondary indexes**: Natural key lookup, FK join optimization, composite filters on common predicates

### Step 6 — Documentation Capture

Read `../ai-native-documentation-design/references/documentation-capture.md` for the full protocol, SQL templates, and ID conventions.

**Module short name:** `DOMAIN` — Decision IDs: `DD-DOMAIN-{NNN}`, Change log: `CL-DOMAIN-{NNN}`, Cookbook: `QC-DOMAIN-{NNN}`

**Typical decisions to capture:**

| Decision | Category | Affects |
|----------|----------|---------|
| Temporal strategy (bi-temporal / Type 2 SCD / current-only) | ARCHITECTURE | All `_H` tables |
| Key strategy (surrogate + natural key) | SCHEMA | All entity tables |
| Delete strategy (soft delete / hard delete) | SCHEMA | All `_H` tables |
| Entity model source (FIBO, BIAN, HL7, custom, etc.) | ARCHITECTURE | All entities |
| Naming conventions (suffixes, prefixes, patterns) | NAMING | All tables and columns |
| Primary index selection per entity | PERFORMANCE | All tables |
| Cross-module FK pattern (generic vs type-safe) | INTEGRATION | All referencing modules |

**Typical glossary terms:** entity key, natural key, bi-temporal, soft delete, current view, history table, reference data, relationship table

**Typical cookbook entries:** current active record query, point-in-time reconstruction, entity relationship traversal

**Required outputs:**
1. `dp_documentation.Module_Registry` INSERT (1 — module registration)
2. `dp_documentation.Design_Decision` INSERTs (minimum 3 — key architectural/schema choices)
3. `dp_documentation.Change_Log` INSERT (1 — CL-DOMAIN-001, INITIAL_RELEASE v1.0.0)
4. `dp_documentation.Business_Glossary` INSERTs (minimum 3 — domain terms introduced)
5. `dp_documentation.Query_Cookbook` INSERTs (minimum 1 — key query patterns)

### Step 7 — Validate Against Checklist
Run the agent-discoverability test and design review checklist in `references/checklists.md`.

---

## Reference Files

Read these when needed — don't load all at once:

| File | Read When |
|------|-----------|
| `references/domain-structure.md` | Writing DDL templates for entities, relationships, reference data, views |
| `references/implementation-guidance.md` | Bi-temporal DML patterns, column optimization, surrogate key sequences, soft delete, indexes |
| `references/checklists.md` | Final review before delivering design |
| `../ai-native-documentation-design/references/documentation-capture.md` | Documentation Capture step — SQL templates and ID conventions |

---

## Quick Reference: Naming Conventions

```
{Entity}_H           -- History/versioned entity table
{Entity}_R           -- Reference/lookup table
{Entity1}{Entity2}_H -- Relationship table
{Entity}_Current     -- View: current active records
{Entity}_Enriched    -- View: with common joins (optional)
{Entity}_AuditTrail  -- View: full history (optional)
```

**Column patterns (apply consistently across ALL entities):**
```
{entity}_key         -- BIGINT surrogate key (FK references use this)
{entity}_id          -- VARCHAR natural/business key
is_current           -- BYTEINT 1/0 current version flag
is_deleted           -- BYTEINT 1/0 soft delete flag
valid_from_dts       -- TIMESTAMP(6) WITH TIME ZONE (bi-temporal: real-world validity)
valid_to_dts         -- TIMESTAMP(6) WITH TIME ZONE
transaction_from_dts -- TIMESTAMP(6) WITH TIME ZONE (bi-temporal: DB recording time)
transaction_to_dts   -- TIMESTAMP(6) WITH TIME ZONE
{type}_code          -- VARCHAR type/status columns (consistent _code suffix)
```

**Standard query pattern (agents learn this once):**
```sql
-- Current active records (all entities)
WHERE is_current = 1 AND is_deleted = 0

-- Via view (preferred for agents)
SELECT * FROM {Entity}_Current WHERE {entity}_id = 'ID'
```

---

## Industry Entity Model Sources

Point the user toward appropriate standards before designing entities:
- **Finance**: FIBO, BIAN, ISO 20022
- **Healthcare**: HL7 FHIR, SNOMED CT, LOINC
- **Retail**: GS1, ARTS
- **Insurance**: ACORD
- **Telecom**: TM Forum SID
- **Enterprise**: iLDM, CDM, Corporate Data Dictionary
- **General**: Schema.org, Dublin Core, SKOS
