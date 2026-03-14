# Design Review Checklists
## AI-Native Documentation Module

---

## 1. Bootstrap Checklist

Before generating DDL for the documentation database, confirm:

- [ ] Database name is `dp_documentation` (shared across all data products)
- [ ] Documentation database is being created BEFORE any other module
- [ ] No cross-database references in the DDL (standalone requirement)
- [ ] All 6 tables include `data_product VARCHAR(100) NOT NULL` as the first business column
- [ ] All 6 tables included: Module_Registry, Design_Decision, Business_Glossary, Query_Cookbook, Implementation_Note, Change_Log
- [ ] Standard views included (unfiltered by data_product — consumers filter)
- [ ] COLLECT STATISTICS includes `data_product` column on every table
- [ ] No seed data (tables start empty — content captured during module design)
- [ ] `data_product = 'ALL'` convention documented in COMMENT ON TABLE

---

## 2. Capture Checklist

When a module skill generates documentation INSERT statements:

- [ ] Documentation SQL written **inline** as the last numbered file in the module directory (e.g., `01-domain/05-documentation.sql`), NOT in a separate documentation batch directory
- [ ] Cross-product standards written to `00-documentation-standards.sql` at root level
- [ ] File header includes `-- Deploy: Phase {N}, after {Module} DDL` comment
- [ ] `data_product` value is consistent across all INSERTs for this data product
- [ ] Module registered in `Module_Registry` with correct `data_product` and version 1.0.0
- [ ] `Module_Registry.version_date` set to the design/deployment date
- [ ] All significant design decisions captured as `Design_Decision` rows
- [ ] Decision IDs follow convention: `DD-{MODULE}-{NNN}` (e.g., DD-GOLD-001)
- [ ] Cross-product standards use `data_product = 'ALL'` and `DD-STD-{NNN}` IDs
- [ ] Each decision has all ADR fields: context, description, alternatives, rationale, consequences
- [ ] `source_module` populated on every row (links record to its module)
- [ ] `module_version` populated on every row (links record to version)
- [ ] `valid_from` set to design/deployment date
- [ ] `valid_to` defaults to `DATE '9999-12-31'`
- [ ] `is_current = 1` on all new records
- [ ] `Change_Log` entry created for initial release (version 1.0.0) with correct `data_product`
- [ ] `Business_Glossary` entries created for domain terms introduced by the module
- [ ] Change log IDs follow convention: `CL-{MODULE}-{NNN}`
- [ ] Glossary terms linked to correct `data_product` and `source_module`

---

## 3. Generation Checklist

When generating Markdown documentation:

- [ ] Data product confirmed: which `data_product` value
- [ ] Module scope confirmed: single module or all modules
- [ ] Version anchor resolved: version number (looked up from Module_Registry) or date
- [ ] All temporal tables queried with correct point-in-time filter:
      `WHERE data_product IN (:dp, 'ALL') AND valid_from <= :as_of_date AND valid_to > :as_of_date`
- [ ] Change_Log queried with `WHERE data_product IN (:dp, 'ALL') AND version_number <= :version`
- [ ] Cross-product standards (`data_product = 'ALL'`) included in results
- [ ] Markdown follows template from `references/markdown-template.md`
- [ ] File naming follows convention (see SKILL.md Workflow 3)
- [ ] Output written to `documentation/{data_product}/` directory at project root
- [ ] Header includes: data product name, module name, version, as-of date, generation date
- [ ] All sections present: Overview, Design Decisions (product-specific + cross-product), Glossary, Cookbook, Notes, Changes
- [ ] Cross-product standards rendered in a separate section before product-specific decisions
- [ ] Empty sections show "No {items} recorded for this module at this version."
- [ ] Cross-module generation includes a table of contents linking to each module section

---

## 4. Structural Checklist

**All Documentation tables:**
- [ ] `GENERATED ALWAYS AS IDENTITY` surrogate key
- [ ] `data_product VARCHAR(100) NOT NULL` as first business column
- [ ] All boolean flags: `BYTEINT NOT NULL DEFAULT value` with `is_` prefix
- [ ] `COMMENT ON TABLE` — explains purpose, multi-product scope, and how agents should use this table
- [ ] `COMMENT ON COLUMN` — all columns have meaningful metadata
- [ ] `created_timestamp` and `updated_timestamp` (where applicable) on every table
- [ ] No `CHAR(1)` or `VARCHAR(1)` boolean columns (must all be `BYTEINT`)

**Temporal columns (all tables except Change_Log):**
- [ ] `source_module VARCHAR(50) NOT NULL` — links record to a module
- [ ] `module_version VARCHAR(20)` — version when record was created
- [ ] `valid_from DATE NOT NULL` — temporal validity start
- [ ] `valid_to DATE DEFAULT DATE '9999-12-31'` — temporal validity end

**Module_Registry:**
- [ ] One row per data_product + module_name + version
- [ ] `is_current` flag: only one row per data_product + module_name has `is_current = 1`
- [ ] `version_date` populated — this is the anchor for version-based queries
- [ ] `module_version` follows semantic versioning (major.minor.patch)

**Design_Decision:**
- [ ] `decision_id` + `decision_version` forms the logical composite key
- [ ] `decision_version` starts at 1, increments for each new version of the same decision
- [ ] `decision_status` constrained to: PROPOSED | ACCEPTED | SUPERSEDED | DEPRECATED
- [ ] `decision_category` constrained to: ARCHITECTURE | SCHEMA | NAMING | PERFORMANCE | SECURITY | INTEGRATION | OPERATIONAL
- [ ] `superseded_by` populated when `decision_status = 'SUPERSEDED'`
- [ ] Version chain: old version has `is_current = 0` and `valid_to` set; new version has `is_current = 1`

**Business_Glossary:**
- [ ] `term_category` constrained to: ENTITY | ATTRIBUTE | METRIC | BUSINESS_RULE | CLASSIFICATION | REFERENCE_CODE
- [ ] `synonyms` populated where abbreviations or alternative names exist
- [ ] `related_table` and `related_column` populated for schema-linked terms

**Query_Cookbook:**
- [ ] `complexity` constrained to: SIMPLE | MODERATE | COMPLEX | ADVANCED
- [ ] `target_tier` constrained to: GOLD | SILVER | BRONZE | CROSS
- [ ] `sql_template` uses `:parameter` syntax for placeholders
- [ ] `parameter_descriptions` in JSON format when parameters exist

**Implementation_Note:**
- [ ] `note_category` constrained to: DEPLOYMENT | WORKAROUND | KNOWN_ISSUE | PERFORMANCE_TIP | OPERATIONAL | SECURITY
- [ ] `severity` constrained to: LOW | MEDIUM | HIGH | CRITICAL (NULL for non-issues)
- [ ] `resolution_status` constrained to: OPEN | IN_PROGRESS | RESOLVED | WONT_FIX

**Change_Log:**
- [ ] `change_type` constrained to: INITIAL_RELEASE | SCHEMA_CHANGE | FEATURE_ADDITION | BUG_FIX | PERFORMANCE | DEPRECATION
- [ ] `change_category` constrained to: BREAKING | NON_BREAKING | ADDITIVE | DEPRECATION
- [ ] `version_number` follows semantic versioning (major.minor.patch)
- [ ] `migration_steps` and `rollback_steps` populated for all changes
- [ ] `related_decision_id` links to a valid `Design_Decision.decision_id` where applicable
- [ ] `deployment_status` constrained to: PLANNED | DEPLOYED | ROLLED_BACK

---

## 5. Content Quality Checklist

**Design Decisions:**
- [ ] All deviations from design standards documented as decisions
- [ ] Each decision has: context, description, alternatives, rationale, and consequences
- [ ] Key architectural choices documented (database layout, temporal strategy, key patterns)
- [ ] Cross-module integration decisions documented (join patterns, key strategies)
- [ ] Cross-product standards captured with `data_product = 'ALL'`

**Business Glossary:**
- [ ] Every entity name has a glossary entry
- [ ] Source system abbreviations mapped to full names
- [ ] Business metrics defined with calculation logic
- [ ] Reference code values documented
- [ ] Enterprise-wide terms use `data_product = 'ALL'`

**Query Cookbook:**
- [ ] At least one recipe per major access pattern (lookup, aggregation, join, analytical)
- [ ] Cross-module join patterns documented
- [ ] Performance notes included for COMPLEX and ADVANCED recipes

**Implementation Notes:**
- [ ] All CRITICAL deployment prerequisites documented
- [ ] Known performance gotchas documented
- [ ] Security considerations documented (PII handling, access control)

**Change Log:**
- [ ] Initial creation of each module logged as version 1.0.0
- [ ] Every subsequent schema change logged with migration and rollback steps
