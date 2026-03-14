---
name: ai-native-documentation-design
description: >
  Design the Documentation module for an AI-Native Data Product on Teradata.
  Shared repository database (dp_documentation) deployed FIRST, covering multiple
  data products. Captures design decisions temporally with version chains.
  Generates human-readable Markdown documentation for any module of any data
  product at any version or point-in-time date. Produces DDL for knowledge tables
  (module_registry, design_decision, business_glossary, query_cookbook,
  implementation_note, change_log), standard discovery views, and cross-module
  capture protocol for other skills to write decisions.
---

# AI-Native Documentation Module Design Skill

## Core Principle (Never Violate)

**Store knowledge and rationale — never duplicate Semantic metadata.**

| Store in Documentation | Keep in Semantic (not here) |
|--------------------------|-------------------------------|
| "We chose bi-temporal because of regulatory replay requirements" | Table/column names, PII flags, join paths |
| "Use LEFT JOIN for Customer → Orders because not all customers have orders" | `table_relationship` rows |
| "`income_bin` is computed as NTILE(5) OVER (ORDER BY income)" | `column_metadata` classification |

**Separation**: Semantic stores *what exists and how it connects*. Documentation stores *why it exists, how to use it, and what changed*.

---

## Role in the Architecture

Documentation is a **shared repository** deployed **first** — before any Domain, Semantic, or other module of any data product. A single `dp_documentation` database serves all data products in the environment.

```
dp_documentation (shared, created once)
  ^  writes decisions (tagged by data_product)
customer_banking skills ────────┤
healthcare skills ──────────────┤
retail skills ──────────────────┤
enterprise-wide standards ──────┘  (data_product = 'ALL')

  ↓  reads for generation (filtered by data_product)
documentation/{data_product}/{module}_v{version}_{date}.md
```

**No cross-database dependencies in core DDL.** The documentation database must be deployable with zero other modules or data products present.

**Multi-product isolation:** Every table carries a `data_product` column. Records with `data_product = 'ALL'` represent cross-product standards that apply everywhere.

---

## Three Workflows

### Workflow 1 — Bootstrap (Create the Documentation Database)

Creates the shared documentation database with all tables and views. Run this **once** per environment — it is not per data product.

**Database name:** `dp_documentation` (fixed, no product name embedded)

**Steps:**
1. Generate CREATE DATABASE `dp_documentation`
2. Generate all 6 tables in order:
   1. `Module_Registry` — register modules across all data products, track versions
   2. `Design_Decision` — ADR-format decisions with version chain
   3. `Business_Glossary` — domain term definitions
   4. `Query_Cookbook` — proven query patterns
   5. `Implementation_Note` — operational knowledge
   6. `Change_Log` — per-module versioned change history
3. Generate standard views
4. Generate COLLECT STATISTICS
5. **No seed data** — tables start empty; content is captured during module design

**Which tables to include:**

| Table | Include When |
|-------|-------------|
| `Module_Registry` | Always — version backbone for point-in-time documentation |
| `Design_Decision` | Always — architectural rationale is always valuable |
| `Change_Log` | Always — version history drives documentation generation |
| `Business_Glossary` | Always — term definitions reduce ambiguity for agents |
| `Query_Cookbook` | Complex data products with multiple query patterns |
| `Implementation_Note` | Complex deployments with operational gotchas |

### Workflow 2 — Capture (Record Decisions During Module Design)

When any module skill for any data product makes a design decision, it generates INSERT statements into `dp_documentation`. Every record is tagged with `data_product` to maintain isolation.

**Output file placement**: Documentation capture SQL is written **inline with each module** as the last numbered script in that module's directory (e.g., `01-domain/05-documentation.sql`). Cross-product standards go to `00-documentation-standards.sql` at root level. See `references/documentation-capture.md` for the full file convention. Never create a separate documentation batch directory.

**Module registration protocol:**
```sql
INSERT INTO dp_documentation.Module_Registry
(data_product, module_name, database_name, module_version, module_purpose,
 key_entities, dependencies, dependents, data_owner, technical_owner,
 version_date, is_current, valid_from, valid_to)
VALUES
('{DATA_PRODUCT}', '{MODULE_NAME}', 'dp_{product}_{module}', '1.0.0',
 '{purpose}', '{entity_list}',
 '{upstream_modules}', '{downstream_modules}',
 '{owner}', '{tech_contact}',
 CURRENT_DATE, 1, CURRENT_DATE, DATE '9999-12-31');
```

**Decision capture protocol:**
```sql
INSERT INTO dp_documentation.Design_Decision
(data_product, decision_id, decision_version, decision_title, decision_description,
 context, alternatives_considered, rationale, consequences,
 decision_status, decision_category,
 source_module, module_version, affects_table,
 decided_by, decided_date, valid_from, valid_to, is_current)
VALUES
('{DATA_PRODUCT}', 'DD-{MODULE}-{NNN}', 1, '{title}', '{description}',
 '{context}', '{alternatives}', '{rationale}', '{consequences}',
 'ACCEPTED', '{category}',
 '{MODULE_NAME}', '{VERSION}', '{table_list}',
 '{agent_or_team}', CURRENT_DATE, CURRENT_DATE, DATE '9999-12-31', 1);
```

**Cross-product decisions** use `data_product = 'ALL'`:
```sql
INSERT INTO dp_documentation.Design_Decision
(data_product, decision_id, decision_version, decision_title, ...)
VALUES
('ALL', 'DD-STD-001', 1, 'All data products use BYTEINT for boolean flags', ...);
```

**Decision ID convention:** `DD-{MODULE}-{NNN}` where MODULE is the short module name (BRONZE, SILVER, GOLD, SEMANTIC, SEARCH, OBSERVABILITY, MEMORY, DOCUMENTATION) and NNN is a zero-padded sequence within that module. For cross-product standards: `DD-STD-{NNN}`.

**Version chain — updating a decision:**
```sql
-- Step 1: Close the current version
UPDATE dp_documentation.Design_Decision
SET valid_to = CURRENT_DATE,
    is_current = 0,
    updated_timestamp = CURRENT_TIMESTAMP(6)
WHERE data_product = '{DATA_PRODUCT}'
  AND decision_id = 'DD-GOLD-001'
  AND is_current = 1;

-- Step 2: Insert new version
INSERT INTO dp_documentation.Design_Decision
(data_product, decision_id, decision_version, decision_title, decision_description,
 context, alternatives_considered, rationale, consequences,
 decision_status, decision_category,
 source_module, module_version, affects_table,
 decided_by, decided_date, valid_from, valid_to, is_current)
VALUES
('{DATA_PRODUCT}', 'DD-GOLD-001', 2, '{updated_title}', '{updated_description}',
 '{updated_context}', '{updated_alternatives}', '{updated_rationale}', '{updated_consequences}',
 'ACCEPTED', '{category}',
 'GOLD', '1.1.0', '{table_list}',
 '{agent_or_team}', CURRENT_DATE, CURRENT_DATE, DATE '9999-12-31', 1);
```

**Change log entry protocol:**
```sql
INSERT INTO dp_documentation.Change_Log
(data_product, change_id, version_number, change_title, change_description,
 change_type, change_category, source_module,
 affects_table, migration_steps, rollback_steps,
 related_decision_id, deployed_date, deployed_by, deployment_status)
VALUES
('{DATA_PRODUCT}', 'CL-{MODULE}-{NNN}', '{VERSION}', '{title}', '{description}',
 '{type}', '{category}', '{MODULE_NAME}',
 '{table_list}', '{migration_sql}', '{rollback_sql}',
 'DD-{MODULE}-{NNN}', CURRENT_DATE, '{deployer}', 'DEPLOYED');
```

### Workflow 3 — Generate (Produce Markdown Documentation)

Generates human-readable Markdown documentation for one or all modules of a specific data product, at a specific version or date.

**Trigger phrases:**
- "Generate documentation for customer_banking Gold at v1.2.0"
- "Generate documentation for all healthcare modules as of 2026-03-01"
- "Generate current documentation for customer_banking"

**Steps:**
1. Determine data product scope: which `data_product` value(s)
2. Determine module scope: single module or all modules
3. Determine version anchor:
   - **By version**: look up `version_date` from `Module_Registry` WHERE `data_product = :dp AND module_version = :version`
   - **By date**: use the date directly as `as_of_date`
   - **Current**: use `CURRENT_DATE` or `is_current = 1`
4. Query all documentation records valid at that point, including:
   - Records WHERE `data_product = :dp` (product-specific)
   - Records WHERE `data_product = 'ALL'` (cross-product standards)
5. Render Markdown using template from `references/markdown-template.md`
6. Write to `documentation/{data_product}/` directory at project root

**File naming convention:**
| Request Type | File Name |
|-------------|-----------|
| Single module by version | `documentation/{data_product}/{module}_v{version}_{YYYYMMDD}.md` |
| Single module by date | `documentation/{data_product}/{module}_asof_{YYYYMMDD}.md` |
| All modules by version | `documentation/{data_product}/all_modules_v{version}_{YYYYMMDD}.md` |
| All modules by date | `documentation/{data_product}/all_modules_asof_{YYYYMMDD}.md` |
| Current (single module) | `documentation/{data_product}/{module}_current.md` |
| Current (all modules) | `documentation/{data_product}/all_modules_current.md` |

Where `{YYYYMMDD}` is the effective date of the version or the requested as-of date. Module names and data product names are lowercase (e.g., `gold`, `customer_banking`).

**During design phase (pre-deployment):**
When the database doesn't exist yet, read INSERT statements from the SQL output files to extract decision records, then render them as Markdown. The SQL files are the source of truth during design.

---

## Table Reference

### Common Column: data_product

Every table includes:
```sql
data_product  VARCHAR(100) NOT NULL
    COMMENT 'Data product this record belongs to (e.g., customer_banking, healthcare). Use ALL for cross-product standards.'
```

This column appears as the **first business column** (after the surrogate key) in every table and is included in the NUPI or secondary index for efficient per-product queries.

### Module_Registry
Tracks all modules across all data products and their version history. One row per data product + module + version.

| Column | Type | Notes |
|--------|------|-------|
| module_registry_key | BIGINT IDENTITY | Surrogate PK |
| data_product | VARCHAR(100) NOT NULL | Data product identifier |
| module_name | VARCHAR(50) NOT NULL | BRONZE, SILVER, GOLD, SEMANTIC, SEARCH, OBSERVABILITY, MEMORY, DOCUMENTATION |
| database_name | VARCHAR(100) NOT NULL | Physical database name |
| module_version | VARCHAR(20) NOT NULL | Semantic version (1.0.0) |
| module_purpose | CLOB NOT NULL | What this module does |
| module_scope | CLOB | In/out of scope |
| key_entities | VARCHAR(500) | Comma-separated primary tables |
| dependencies | VARCHAR(500) | Upstream module dependencies |
| dependents | VARCHAR(500) | Downstream consumers |
| data_owner | VARCHAR(100) | Business owner |
| technical_owner | VARCHAR(100) | Technical contact |
| refresh_frequency | VARCHAR(50) | Refresh cadence |
| version_date | DATE NOT NULL | Date this version was established |
| is_current | BYTEINT NOT NULL DEFAULT 1 | 1 = current version, 0 = historical |
| valid_from | DATE NOT NULL | Temporal validity start |
| valid_to | DATE DEFAULT '9999-12-31' | Temporal validity end |
| created_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record creation |
| updated_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record last update |

### Design_Decision
Architecture Decision Records with version chain. Same `decision_id` can have multiple rows with incrementing `decision_version`.

| Column | Type | Notes |
|--------|------|-------|
| decision_key | BIGINT IDENTITY | Surrogate PK |
| data_product | VARCHAR(100) NOT NULL | Data product (or ALL for cross-product) |
| decision_id | VARCHAR(50) NOT NULL | Human-readable ID (DD-GOLD-001) |
| decision_version | INTEGER NOT NULL DEFAULT 1 | Version of this decision |
| decision_title | VARCHAR(200) NOT NULL | Short title |
| decision_description | CLOB | Full description |
| context | CLOB | What prompted this decision |
| alternatives_considered | CLOB | Other options evaluated |
| rationale | CLOB | Why this was chosen |
| consequences | CLOB | Trade-offs and impacts |
| decision_status | VARCHAR(20) NOT NULL | PROPOSED, ACCEPTED, SUPERSEDED, DEPRECATED |
| decision_category | VARCHAR(50) NOT NULL | ARCHITECTURE, SCHEMA, NAMING, PERFORMANCE, SECURITY, INTEGRATION, OPERATIONAL |
| source_module | VARCHAR(50) NOT NULL | Module that created this decision |
| module_version | VARCHAR(20) | Module version when decided |
| affects_table | VARCHAR(200) | Tables affected |
| decided_by | VARCHAR(100) | Who decided |
| decided_date | DATE | When decided |
| superseded_by | VARCHAR(50) | Replacement decision_id if SUPERSEDED |
| valid_from | DATE NOT NULL | Temporal validity start |
| valid_to | DATE DEFAULT '9999-12-31' | Temporal validity end |
| is_current | BYTEINT NOT NULL DEFAULT 1 | 1 = current version, 0 = historical |
| created_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record creation |
| updated_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record last update |

### Business_Glossary
Domain term definitions. Each term is linked to a data product and source module, versioned temporally.

| Column | Type | Notes |
|--------|------|-------|
| glossary_key | BIGINT IDENTITY | Surrogate PK |
| data_product | VARCHAR(100) NOT NULL | Data product (or ALL for enterprise terms) |
| term | VARCHAR(200) NOT NULL | Business term being defined |
| term_category | VARCHAR(50) NOT NULL | ENTITY, ATTRIBUTE, METRIC, BUSINESS_RULE, CLASSIFICATION, REFERENCE_CODE |
| definition | CLOB NOT NULL | Plain-language definition |
| business_context | CLOB | How this term is used |
| synonyms | VARCHAR(500) | Comma-separated alternate names |
| related_terms | VARCHAR(500) | Comma-separated related terms |
| related_table | VARCHAR(200) | Table where physically implemented |
| related_column | VARCHAR(200) | Column where physically implemented |
| source_module | VARCHAR(50) NOT NULL | Module this term belongs to |
| module_version | VARCHAR(20) | Module version when term was defined |
| is_active | BYTEINT NOT NULL DEFAULT 1 | 1 = active, 0 = retired |
| valid_from | DATE NOT NULL | Temporal validity start |
| valid_to | DATE DEFAULT '9999-12-31' | Temporal validity end |
| created_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record creation |
| updated_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record last update |

### Query_Cookbook
Tested, reusable query patterns. Versioned per data product and source module.

| Column | Type | Notes |
|--------|------|-------|
| recipe_key | BIGINT IDENTITY | Surrogate PK |
| data_product | VARCHAR(100) NOT NULL | Data product this recipe belongs to |
| recipe_id | VARCHAR(50) NOT NULL | Human-readable ID (QC-GOLD-001) |
| recipe_title | VARCHAR(200) NOT NULL | Short title |
| recipe_description | CLOB NOT NULL | What this query does |
| use_case | VARCHAR(200) NOT NULL | Business scenario addressed |
| target_tier | VARCHAR(10) NOT NULL | GOLD, SILVER, BRONZE, CROSS |
| sql_template | CLOB NOT NULL | SQL with :parameter placeholders |
| parameter_descriptions | CLOB | JSON describing parameters |
| performance_notes | CLOB | PI alignment, cost, optimization tips |
| complexity | VARCHAR(20) NOT NULL | SIMPLE, MODERATE, COMPLEX, ADVANCED |
| source_module | VARCHAR(50) NOT NULL | Module this recipe belongs to |
| module_version | VARCHAR(20) | Module version when recipe was created |
| is_active | BYTEINT NOT NULL DEFAULT 1 | 1 = active, 0 = retired |
| valid_from | DATE NOT NULL | Temporal validity start |
| valid_to | DATE DEFAULT '9999-12-31' | Temporal validity end |
| created_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record creation |
| updated_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record last update |

### Implementation_Note
Operational knowledge, workarounds, known issues. Versioned per data product and source module.

| Column | Type | Notes |
|--------|------|-------|
| note_key | BIGINT IDENTITY | Surrogate PK |
| data_product | VARCHAR(100) NOT NULL | Data product this note belongs to |
| note_id | VARCHAR(50) NOT NULL | Human-readable ID (IN-GOLD-001) |
| note_title | VARCHAR(200) NOT NULL | Short title |
| note_content | CLOB NOT NULL | Full note content |
| note_category | VARCHAR(50) NOT NULL | DEPLOYMENT, WORKAROUND, KNOWN_ISSUE, PERFORMANCE_TIP, OPERATIONAL, SECURITY |
| severity | VARCHAR(20) | LOW, MEDIUM, HIGH, CRITICAL (NULL for non-issues) |
| affects_table | VARCHAR(200) | Tables this note applies to |
| resolution_status | VARCHAR(20) | OPEN, IN_PROGRESS, RESOLVED, WONT_FIX |
| resolution_notes | CLOB | How issue was resolved |
| source_module | VARCHAR(50) NOT NULL | Module this note belongs to |
| module_version | VARCHAR(20) | Module version when note was created |
| is_active | BYTEINT NOT NULL DEFAULT 1 | 1 = active, 0 = retired |
| valid_from | DATE NOT NULL | Temporal validity start |
| valid_to | DATE DEFAULT '9999-12-31' | Temporal validity end |
| created_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record creation |
| updated_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record last update |

### Change_Log
Versioned history of changes per data product and module. Each row IS a point-in-time event.

| Column | Type | Notes |
|--------|------|-------|
| change_key | BIGINT IDENTITY | Surrogate PK |
| data_product | VARCHAR(100) NOT NULL | Data product this change belongs to |
| change_id | VARCHAR(50) NOT NULL | Human-readable ID (CL-GOLD-001) |
| version_number | VARCHAR(20) NOT NULL | Semantic version (1.0.0, 1.1.0) |
| change_title | VARCHAR(200) NOT NULL | Short title |
| change_description | CLOB NOT NULL | Full description |
| change_type | VARCHAR(30) NOT NULL | INITIAL_RELEASE, SCHEMA_CHANGE, FEATURE_ADDITION, BUG_FIX, PERFORMANCE, DEPRECATION |
| change_category | VARCHAR(50) NOT NULL | BREAKING, NON_BREAKING, ADDITIVE, DEPRECATION |
| source_module | VARCHAR(50) NOT NULL | Module this change affects |
| affects_table | VARCHAR(200) | Specific tables affected |
| migration_steps | CLOB | SQL to apply this change |
| rollback_steps | CLOB | SQL to revert this change |
| related_decision_id | VARCHAR(50) | Links to Design_Decision.decision_id |
| deployed_date | DATE | Date deployed |
| deployed_by | VARCHAR(100) | Who deployed |
| deployment_status | VARCHAR(20) NOT NULL | PLANNED, DEPLOYED, ROLLED_BACK |
| created_timestamp | TIMESTAMP(6) WITH TIME ZONE | Record creation |

---

## Temporal Query Patterns

All temporal queries must account for both product-specific and cross-product (`ALL`) records.

**All decisions for a data product + module at a specific version:**
```sql
SELECT dd.*
FROM dp_documentation.Design_Decision dd
WHERE dd.data_product IN (:data_product, 'ALL')
  AND dd.source_module = :module_name
  AND dd.valid_from <= (
      SELECT mr.version_date
      FROM dp_documentation.Module_Registry mr
      WHERE mr.data_product = :data_product
        AND mr.module_name = :module_name
        AND mr.module_version = :version
  )
  AND dd.valid_to > (
      SELECT mr.version_date
      FROM dp_documentation.Module_Registry mr
      WHERE mr.data_product = :data_product
        AND mr.module_name = :module_name
        AND mr.module_version = :version
  );
```

**All decisions for a data product + module at a specific date:**
```sql
SELECT * FROM dp_documentation.Design_Decision
WHERE data_product IN (:data_product, 'ALL')
  AND source_module = :module_name
  AND valid_from <= :as_of_date
  AND valid_to > :as_of_date;
```

**All decisions for a data product (all modules) at a date:**
```sql
SELECT * FROM dp_documentation.Design_Decision
WHERE data_product IN (:data_product, 'ALL')
  AND valid_from <= :as_of_date
  AND valid_to > :as_of_date;
```

**Same pattern applies to all temporal tables** (Business_Glossary, Query_Cookbook, Implementation_Note). Always include `data_product IN (:dp, 'ALL')`.

**Change log up to a version for a data product + module:**
```sql
SELECT * FROM dp_documentation.Change_Log
WHERE data_product IN (:data_product, 'ALL')
  AND source_module = :module_name
  AND version_number <= :version
ORDER BY version_number, created_timestamp;
```

---

## Standard Views

All views include `data_product` in their output for filtering. Views do NOT filter by data_product so they serve all products.

| View | Purpose |
|------|---------|
| `v_Current_Decisions` | All current (is_current=1) non-superseded decisions, all products |
| `v_Module_Registry_Current` | Current version of each module across all products |
| `v_Glossary_Active` | Active glossary terms (is_active=1, valid_to = 9999-12-31) |
| `v_Cookbook_Active` | Active recipes with complexity ranking |
| `v_Issues_Open` | Open issues ordered by severity |
| `v_Change_History` | Full change log ordered by version desc |
| `v_Documentation_Search` | Unified text search across all documentation tables |

Consumers filter by `WHERE data_product IN ('{my_product}', 'ALL')`.

---

## Cross-Module Capture Protocol

When another module skill (Domain, Semantic, Search, etc.) makes a design decision during its design workflow, it should generate INSERT statements into `dp_documentation` following the capture protocol above.

**Each module skill is responsible for:**
1. Registering itself in `Module_Registry` with the correct `data_product`
2. Generating `Design_Decision` INSERTs for every significant design choice
3. Generating a `Change_Log` entry for the initial release (version 1.0.0)
4. Generating `Business_Glossary` entries for domain terms it introduces
5. Generating `Query_Cookbook` entries for key query patterns it supports

**Decision categories by module type:**

| Module | Typical Decision Categories |
|--------|-----------------------------|
| Bronze | ARCHITECTURE (source mapping), SCHEMA (raw table structure) |
| Silver | SCHEMA (entity unification), NAMING (column standardisation) |
| Gold | ARCHITECTURE (materialisation strategy), PERFORMANCE (PI choice) |
| Semantic | INTEGRATION (relationship mapping), NAMING (metadata standards) |
| Search | ARCHITECTURE (vector strategy), PERFORMANCE (index choice) |
| Observability | OPERATIONAL (monitoring thresholds), INTEGRATION (lineage scope) |
| Memory | ARCHITECTURE (session strategy), SECURITY (privacy scoping) |

**Implementation**: Each module skill (Domain, Semantic, Prediction, Search, Observability, Memory) references the shared `references/documentation-capture.md` with the full protocol, SQL templates, ID conventions, and **output file convention**. A dedicated "Documentation Capture" workflow step in each skill instructs the agent to review design decisions and generate INSERTs, writing the output as the last numbered file in the module directory (e.g., `01-domain/05-documentation.sql`). Cross-product standards are written to `00-documentation-standards.sql` at root level.

**Cross-product standards** (data_product = 'ALL') are typically captured by the Documentation skill itself or by an architecture governance process:
- Boolean conventions (BYTEINT, is_ prefix)
- Naming standards (snake_case, _H suffix for history)
- Temporal patterns (valid_from/valid_to conventions)
- Key patterns (surrogate + natural key strategy)

---

## Boolean Conventions

All boolean flag columns follow standard conventions:
- **Type**: `BYTEINT NOT NULL DEFAULT value`
- **Naming**: `is_` prefix
- **Values**: `1` = true/yes/active, `0` = false/no/inactive

Applied in Documentation:
```
is_current          BYTEINT NOT NULL DEFAULT 1  -- design_decision, module_registry
is_active           BYTEINT NOT NULL DEFAULT 1  -- business_glossary, query_cookbook, implementation_note
```

---

## Decision Status Values

Standard lifecycle for `design_decision.decision_status`:

| Status | Meaning |
|--------|---------|
| `PROPOSED` | Under discussion, not yet agreed |
| `ACCEPTED` | Agreed and being implemented |
| `SUPERSEDED` | Replaced by a newer decision (link via `superseded_by`) |
| `DEPRECATED` | No longer relevant but kept for historical context |

---

## Reference Files

| File | Read When |
|------|-----------|
| `references/checklists.md` | Final review — bootstrap, capture, generation, and structural quality |
| `references/markdown-template.md` | Generating Markdown documentation output — use as the rendering template |
| `references/documentation-capture.md` | Cross-module capture — SQL templates, ID conventions, minimum requirements |
