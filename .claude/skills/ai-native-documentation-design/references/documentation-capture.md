# Documentation Capture Protocol

## Purpose

This file is the single-source reference for all module skills (Domain, Semantic, Prediction, Search, Observability, Memory) to generate INSERT statements into `dp_documentation` during their design workflow. Every module must capture its design decisions before completing.

---

## Module Short Names

| Module | Short Name | ID Prefix |
|--------|-----------|-----------|
| Domain | DOMAIN | DD-DOMAIN- |
| Semantic | SEMANTIC | DD-SEMANTIC- |
| Prediction | PREDICTION | DD-PREDICTION- |
| Search | SEARCH | DD-SEARCH- |
| Observability | OBSERVABILITY | DD-OBSERVABILITY- |
| Memory | MEMORY | DD-MEMORY- |
| Documentation | DOCUMENTATION | DD-DOCUMENTATION- |
| Cross-product | STD | DD-STD- |

---

## ID Conventions

- **Design Decisions:** `DD-{MODULE}-{NNN}` — zero-padded sequence within module (e.g., `DD-DOMAIN-001`)
- **Change Log:** `CL-{MODULE}-{NNN}` — (e.g., `CL-DOMAIN-001`)
- **Query Cookbook:** `QC-{MODULE}-{NNN}` — (e.g., `QC-DOMAIN-001`)
- **Cross-product standards:** `DD-STD-{NNN}` — with `data_product = 'ALL'`; capture once (typically by first module or Documentation skill)

---

## Decision Categories

Use these standard values for `decision_category`:

| Category | Use For |
|----------|---------|
| `ARCHITECTURE` | High-level structural choices (storage patterns, module layout) |
| `SCHEMA` | Table/column design decisions |
| `NAMING` | Naming conventions, prefixes, suffixes |
| `PERFORMANCE` | Primary index, partitioning, indexing choices |
| `SECURITY` | Privacy scoping, PII handling, access control |
| `INTEGRATION` | Cross-module FK patterns, join strategies |
| `OPERATIONAL` | Retention policies, monitoring thresholds, refresh strategies |

---

## Minimum Capture Requirements

Every module must generate at least the following:

| Record Type | Table | Minimum | Notes |
|-------------|-------|---------|-------|
| Module_Registry | `dp_documentation.Module_Registry` | 1 | Exactly one registration per module |
| Design_Decision | `dp_documentation.Design_Decision` | 3 | Key architectural/schema choices |
| Change_Log | `dp_documentation.Change_Log` | 1 | Initial release v1.0.0 |
| Business_Glossary | `dp_documentation.Business_Glossary` | 3 | Domain terms introduced by this module |
| Query_Cookbook | `dp_documentation.Query_Cookbook` | 1 | Key query patterns for this module |

---

## SQL Templates

### Module_Registry INSERT

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

### Design_Decision INSERT

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
 '{MODULE_NAME}', '1.0.0', '{table_list}',
 '{agent_or_team}', CURRENT_DATE, CURRENT_DATE, DATE '9999-12-31', 1);
```

**Required ADR fields** (every decision must have all of these populated):
- `context` — Why this decision was needed
- `alternatives_considered` — What other options existed
- `rationale` — Why this alternative was chosen
- `consequences` — What follows from this decision (positive and negative)

### Change_Log INSERT

```sql
INSERT INTO dp_documentation.Change_Log
(data_product, change_id, version_number, change_title, change_description,
 change_type, change_category, source_module,
 affects_table, migration_steps, rollback_steps,
 related_decision_id, deployed_date, deployed_by, deployment_status)
VALUES
('{DATA_PRODUCT}', 'CL-{MODULE}-001', '1.0.0', '{MODULE_NAME} initial release',
 '{description of all tables and views created}',
 'INITIAL_RELEASE', 'SCHEMA', '{MODULE_NAME}',
 '{table_list}', '{migration_sql}', '{rollback_sql}',
 'DD-{MODULE}-001', CURRENT_DATE, '{deployer}', 'DEPLOYED');
```

### Business_Glossary INSERT

```sql
INSERT INTO dp_documentation.Business_Glossary
(data_product, term, term_category, definition, business_context,
 synonyms, related_terms, related_table, related_column,
 source_module, module_version, is_active, valid_from, valid_to)
VALUES
('{DATA_PRODUCT}', '{term}', '{ENTITY|ATTRIBUTE|METRIC|BUSINESS_RULE|CLASSIFICATION|REFERENCE_CODE}',
 '{plain-language definition}', '{how this term is used in the data product}',
 '{comma-separated synonyms}', '{comma-separated related terms}',
 '{table_name}', '{column_name}',
 '{MODULE_NAME}', '1.0.0', 1, CURRENT_DATE, DATE '9999-12-31');
```

### Query_Cookbook INSERT

```sql
INSERT INTO dp_documentation.Query_Cookbook
(data_product, recipe_id, recipe_title, recipe_description,
 use_case, target_tier, sql_template, parameter_descriptions,
 performance_notes, complexity,
 source_module, module_version, is_active, valid_from, valid_to)
VALUES
('{DATA_PRODUCT}', 'QC-{MODULE}-001', '{title}', '{what this query does}',
 '{business scenario}', '{GOLD|SILVER|BRONZE|CROSS}',
 '{SQL with :parameter placeholders}', '{JSON parameter descriptions}',
 '{PI alignment, cost, optimization tips}', '{SIMPLE|MODERATE|COMPLEX|ADVANCED}',
 '{MODULE_NAME}', '1.0.0', 1, CURRENT_DATE, DATE '9999-12-31');
```

### Version Chain — Updating a Decision

```sql
-- Step 1: Close the current version
UPDATE dp_documentation.Design_Decision
SET valid_to = CURRENT_DATE,
    is_current = 0,
    updated_timestamp = CURRENT_TIMESTAMP(6)
WHERE data_product = '{DATA_PRODUCT}'
  AND decision_id = 'DD-{MODULE}-{NNN}'
  AND is_current = 1;

-- Step 2: Insert new version
INSERT INTO dp_documentation.Design_Decision
(data_product, decision_id, decision_version, decision_title, decision_description,
 context, alternatives_considered, rationale, consequences,
 decision_status, decision_category,
 source_module, module_version, affects_table,
 decided_by, decided_date, valid_from, valid_to, is_current)
VALUES
('{DATA_PRODUCT}', 'DD-{MODULE}-{NNN}', 2, '{updated_title}', '{updated_description}',
 '{updated_context}', '{updated_alternatives}', '{updated_rationale}', '{updated_consequences}',
 'ACCEPTED', '{category}',
 '{MODULE_NAME}', '{NEW_VERSION}', '{table_list}',
 '{agent_or_team}', CURRENT_DATE, CURRENT_DATE, DATE '9999-12-31', 1);
```

---

## Temporal Field Standards

All documentation INSERTs must use these consistent temporal values:

- `valid_from = CURRENT_DATE`
- `valid_to = DATE '9999-12-31'`
- `is_current = 1` (for Design_Decision, Module_Registry)
- `is_active = 1` (for Business_Glossary, Query_Cookbook)
- `decision_status = 'ACCEPTED'` (for new decisions)

---

## Cross-Product Standards Note

Decisions with `data_product = 'ALL'` and ID prefix `DD-STD-{NNN}` capture enterprise-wide standards:
- Boolean conventions (BYTEINT, `is_` prefix)
- Naming standards (snake_case, `_H` suffix for history)
- Temporal patterns (valid_from/valid_to conventions)
- Key patterns (surrogate + natural key strategy)

These should only be captured **once** — typically by the first module designed or by the Documentation skill itself. Do not duplicate `DD-STD-*` decisions across modules.
