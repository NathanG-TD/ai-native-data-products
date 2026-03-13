# Design Review Checklists
## AI-Native Domain Module

Use before delivering any Domain module design. Run in order.

---

## 1. Designer Requirements Checklist

Before writing DDL, confirm the following have been gathered:

- [ ] Entity model source identified (iLDM, FIBO, BIAN, HL7, custom, etc.)
- [ ] Each entity's business purpose documented in plain language
- [ ] Natural/business key(s) for each entity identified
- [ ] Entity-to-entity relationships mapped
- [ ] Temporal strategy chosen and justified (bi-temporal / Type 2 SCD / current-only)
- [ ] Column set tier chosen and justified (Core / Core+FK / Core+Extended)
- [ ] Source system(s) identified per entity
- [ ] Update frequency documented
- [ ] Retention policy defined
- [ ] Data quality rules defined (even if basic)
- [ ] Primary index selection justified per entity
- [ ] Partitioning strategy justified where applicable

---

## 2. Structural Completeness Checklist

For each Domain entity table (`_H`):

- [ ] `{entity}_key` — BIGINT surrogate key present
- [ ] `{entity}_id` — Natural/business key present
- [ ] Temporal tracking supports point-in-time reconstruction
- [ ] `is_current` column present (BYTEINT)
- [ ] `is_deleted` column present (BYTEINT)
- [ ] `PRIMARY INDEX` (non-unique for temporal tables)
- [ ] At minimum: `created_by_user_id`, `source_system_id` audit columns
- [ ] `COMMENT ON TABLE` — business purpose (not technical)
- [ ] `COMMENT ON COLUMN` — every column has meaningful metadata

For reference data tables (`_R`):

- [ ] `{reference}_code` — short identifier present
- [ ] `short_description` and `long_description` present
- [ ] `effective_date` and `expiration_date` present
- [ ] `is_current` present
- [ ] `UNIQUE PRIMARY INDEX ({ref}_code, effective_date)` — appropriate for reference data

For relationship tables:

- [ ] Both FKs (`{entity1}_key`, `{entity2}_key`) present with descriptive names
- [ ] Same temporal pattern as entity tables (consistency for agent learning)
- [ ] `PRIMARY INDEX ({entity1}_key)` for co-location with primary entity

---

## 3. Standard Views Checklist

- [ ] `{Entity}_Current` view created for every entity
- [ ] `COMMENT ON VIEW` explains filtering logic for every view
- [ ] Enriched views created where common join patterns exist
- [ ] Views tested — return expected results with standard filter pattern

---

## 4. Naming Consistency Checklist

Apply these checks across ALL entities (consistency is the goal — agents learn patterns):

- [ ] All surrogate keys follow `{entity}_key` pattern
- [ ] All natural keys follow `{entity}_id` pattern
- [ ] Temporal column names identical across all entities
- [ ] Boolean flags use `is_` prefix consistently
- [ ] Type/status columns use `_code` suffix consistently
- [ ] Foreign keys are descriptive (no `fk1`, `fk2`)
- [ ] Table suffixes used consistently (`_H`, `_R`, no suffix for views)
- [ ] Abbreviations documented in Semantic module

---

## 5. Agent Discoverability Test

**Can an agent that has never seen your entities:**

1. **Discover** what entities exist?
   - `SELECT TableName, CommentString FROM DBC.TablesV WHERE DatabaseName = '...'`
   - → Table names and comments return meaningful results ✅

2. **Understand** what each entity represents?
   - `SELECT ColumnName, CommentString FROM DBC.ColumnsV WHERE TableName = '...'`
   - → All columns have meaningful, non-redundant comments ✅

3. **Find current active records** without special knowledge?
   - `WHERE is_current = 1 AND is_deleted = 0` works on every entity ✅
   - `{Entity}_Current` view available for every entity ✅

4. **Navigate relationships?**
   - FK column names tell the agent exactly what they reference ✅
   - e.g. `party_key` not `fk1`

5. **Generate valid queries** for any entity programmatically?
   - Same key/flag/temporal column names across all entities ✅
   - Patterns documented in Semantic module ✅

**If all 5 → Agent-discoverable ✅**

---

## 6. Cross-Module Integration Checklist

- [ ] FK pattern chosen (Pattern A generic / Pattern B specific) and applied consistently within each referencing module
- [ ] Domain attributes accessed via JOIN, never copied into other modules
- [ ] Standard view (`{Entity}_Current`) used as join target in examples
- [ ] Temporal join patterns documented (for point-in-time feature computation)

---

## 7. Physical Design Checklist

- [ ] Temporal tables use non-unique PRIMARY INDEX (NUPI)
- [ ] Reference tables use UNIQUE PRIMARY INDEX on (code, effective_date)
- [ ] Secondary indexes justified — not created speculatively
- [ ] Statistics collection scheduled and documented
- [ ] Compression applied only to large text columns (>500 chars)
- [ ] Storage estimates calculated for first 12 months
- [ ] Query performance tested against physical design

---

## 8. Documentation Capture Checklist

- [ ] `dp_documentation.Module_Registry` INSERT generated (module_name = 'DOMAIN', version 1.0.0)
- [ ] `dp_documentation.Design_Decision` INSERTs generated (minimum 3 decisions)
- [ ] Decision IDs follow `DD-DOMAIN-{NNN}` convention
- [ ] Every decision has all ADR fields: context, alternatives_considered, rationale, consequences
- [ ] Decision categories from standard set: ARCHITECTURE | SCHEMA | NAMING | PERFORMANCE | SECURITY | INTEGRATION | OPERATIONAL
- [ ] All decisions: `decision_status = 'ACCEPTED'`, `is_current = 1`
- [ ] `dp_documentation.Change_Log` INSERT generated (CL-DOMAIN-001, INITIAL_RELEASE)
- [ ] Change log includes `migration_steps` and `rollback_steps`
- [ ] `dp_documentation.Business_Glossary` INSERTs generated (minimum 3 terms)
- [ ] `dp_documentation.Query_Cookbook` INSERTs generated (minimum 1 recipe)
- [ ] All INSERTs use consistent `data_product` value
- [ ] All temporal fields: `valid_from = CURRENT_DATE`, `valid_to = DATE '9999-12-31'`
- [ ] `source_module` and `module_version` populated on every row

---

## 9. Final Pre-Delivery Checklist

- [ ] All entities follow same structural patterns (no special cases)
- [ ] Metadata quality validated — no "column_name" == column purpose comments
- [ ] Semantic module entries planned for naming standards and entity metadata
- [ ] Implementation order documented: Domain first, then Semantic, then other modules
- [ ] Deviations from advocated standards documented with rationale
- [ ] Documentation capture complete — all dp_documentation INSERTs generated
