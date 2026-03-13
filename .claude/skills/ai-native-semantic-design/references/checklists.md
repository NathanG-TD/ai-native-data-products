# Design Review Checklists
## AI-Native Semantic Module

---

## 1. Designer Requirements Checklist

Before writing DDL, confirm:

- [ ] Data product name and database layout confirmed (SEPARATE_DB vs SINGLE_DB_PREFIX)
- [ ] All Domain module entities and their relationships captured
- [ ] All modules identified: which are DEPLOYED, PLANNED, or DEPRECATED
- [ ] Enterprise naming conventions and abbreviations gathered
- [ ] PII / sensitive / classification policy confirmed
- [ ] Optional tables decision made: ontology? business_rule? data_contract_catalog?

---

## 2. Table Completeness Checklist

**Core tables (always required):**
- [ ] `data_product_map` — DDL created
- [ ] `entity_metadata` — DDL created
- [ ] `column_metadata` — DDL created
- [ ] `naming_standard` — DDL created
- [ ] `table_relationship` — DDL created

**Optional tables (as decided):**
- [ ] `ontology` — DDL created (if formal taxonomy required)
- [ ] `business_rule` — DDL created (if rules must be queryable)
- [ ] `data_contract_catalog` — DDL created (if ODCS contracts in use)

---

## 3. View Completeness Checklist

- [ ] `v_relationship_paths` — recursive CTE created (use exact template from `references/semantic-views.md`)
- [ ] `v_entity_catalog` — entity + module join view created
- [ ] `v_entity_schema` — column metadata view created
- [ ] `COMMENT ON VIEW` added to all views

---

## 4. Data Population Checklist

Populate in order: `data_product_map` → `entity_metadata` → `naming_standard` → `column_metadata` → `table_relationship`

- [ ] `data_product_map`: one row per module (including Semantic itself)
- [ ] `data_product_map`: `primary_tables` and `primary_views` filled for all DEPLOYED modules
- [ ] `entity_metadata`: one row per table across all modules (not just Domain)
- [ ] `entity_metadata`: `view_name` populated for all Domain entities
- [ ] `entity_metadata`: `surrogate_key_column` and `natural_key_column` populated for all entities with those columns
- [ ] `entity_metadata`: `temporal_pattern` populated (BI_TEMPORAL / TYPE_2_SCD / NONE)
- [ ] `naming_standard`: all suffixes documented (`_H`, `_R`, `_key`, `_id`, `_dts`, `_code`, `_amt`)
- [ ] `naming_standard`: all boolean prefixes documented (`is_`)
- [ ] `naming_standard`: FK pattern documented
- [ ] `naming_standard`: enterprise-specific abbreviations added
- [ ] `column_metadata`: all PII columns registered (`is_pii = 'Y'`)
- [ ] `column_metadata`: all sensitive columns registered (`is_sensitive = 'Y'`)
- [ ] `column_metadata`: `allowed_values_json` populated for constrained code columns
- [ ] `table_relationship`: all FK joins registered (Child→Parent direction)
- [ ] `table_relationship`: all hierarchy relationships registered (self-joins)
- [ ] `table_relationship`: all associative/bridge table joins registered (both sides)
- [ ] `table_relationship`: cross-module relationships registered (e.g., Prediction→Domain FKs)

---

## 5. Agent Discovery Test

Run these queries against a populated Semantic module. All should return meaningful results.

```sql
-- Test 1: Module discovery
SELECT module_name, database_name, deployment_status
FROM Semantic.data_product_map WHERE is_active = 'Y';
-- Expected: one row per module with correct database names

-- Test 2: Entity discovery
SELECT module_name, entity_name, table_name, view_name
FROM Semantic.v_entity_catalog;
-- Expected: all tables across all modules with views populated for Domain entities

-- Test 3: Naming interpretation
SELECT standard_value, meaning FROM Semantic.naming_standard
WHERE standard_type = 'SUFFIX' AND is_active = 'Y';
-- Expected: _H, _R, _key, _id, _dts, _code, _Current at minimum

-- Test 4: Direct join path (1-hop)
SELECT hop_count, path_joins
FROM Semantic.v_relationship_paths
WHERE source_table = '{ChildTable_H}'
  AND target_table = '{ParentTable_H}'
ORDER BY hop_count QUALIFY ROW_NUMBER() OVER (ORDER BY hop_count) = 1;
-- Expected: hop_count = 1, valid JOIN syntax

-- Test 5: Multi-hop path (2+ hops)
SELECT hop_count, path_tables, path_joins
FROM Semantic.v_relationship_paths
WHERE source_table = '{TableA}'
  AND target_table = '{TableC}'   -- A→B→C via bridge
ORDER BY hop_count QUALIFY ROW_NUMBER() OVER (ORDER BY hop_count) = 1;
-- Expected: hop_count = 2, path_tables shows intermediate table

-- Test 6: Reverse traversal
SELECT hop_count, path_joins
FROM Semantic.v_relationship_paths
WHERE source_table = '{ParentTable_H}'   -- reverse direction from Test 4
  AND target_table = '{ChildTable_H}'
ORDER BY hop_count QUALIFY ROW_NUMBER() OVER (ORDER BY hop_count) = 1;
-- Expected: returns result (bidirectional support working)

-- Test 7: PII/governance check
SELECT table_name, column_name, data_classification
FROM Semantic.v_entity_schema
WHERE is_pii = 'Y';
-- Expected: all PII columns identified
```

**Pass criteria**: All 7 tests return expected results → Semantic module is agent-ready ✅

---

## 6. Documentation Capture Checklist

- [ ] `dp_documentation.Module_Registry` INSERT generated (module_name = 'SEMANTIC', version 1.0.0)
- [ ] `dp_documentation.Design_Decision` INSERTs generated (minimum 3 decisions)
- [ ] Decision IDs follow `DD-SEMANTIC-{NNN}` convention
- [ ] Every decision has all ADR fields: context, alternatives_considered, rationale, consequences
- [ ] Decision categories from standard set: ARCHITECTURE | SCHEMA | NAMING | PERFORMANCE | SECURITY | INTEGRATION | OPERATIONAL
- [ ] All decisions: `decision_status = 'ACCEPTED'`, `is_current = 1`
- [ ] `dp_documentation.Change_Log` INSERT generated (CL-SEMANTIC-001, INITIAL_RELEASE)
- [ ] Change log includes `migration_steps` and `rollback_steps`
- [ ] `dp_documentation.Business_Glossary` INSERTs generated (minimum 3 terms)
- [ ] `dp_documentation.Query_Cookbook` INSERTs generated (minimum 1 recipe)
- [ ] All INSERTs use consistent `data_product` value
- [ ] All temporal fields: `valid_from = CURRENT_DATE`, `valid_to = DATE '9999-12-31'`
- [ ] `source_module` and `module_version` populated on every row

---

## 7. Final Pre-Delivery Checklist

- [ ] All core tables created with `COMMENT ON TABLE` and `COMMENT ON COLUMN`
- [ ] `v_relationship_paths` uses the exact validated CTE template — not modified
- [ ] Every Domain entity has a row in `entity_metadata`
- [ ] Every Semantic table has a row in `entity_metadata` (Semantic describes itself)
- [ ] Population order documented for ETL/ELT processes
- [ ] `data_product_map` updated as modules move from PLANNED to DEPLOYED
- [ ] Test queries above run successfully on target Teradata instance
- [ ] Documentation capture complete — all dp_documentation INSERTs generated
