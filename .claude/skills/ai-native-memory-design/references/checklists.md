# Design Review Checklists
## AI-Native Memory Module

---

## 1. Designer Requirements Checklist

Before writing DDL, confirm:

- [ ] Agent types and their session patterns identified
- [ ] Learning categories needed: QUERY_OPTIMIZATION | FEATURE_SELECTION | ERROR_HANDLING | other
- [ ] Privacy scope levels in use: USER | TEAM | ORGANIZATION | AGENT
- [ ] User identifier format agreed (must be an ID — no PII stored directly)
- [ ] Retention policy per table defined
- [ ] Observability module deployed? (Memory feed pattern requires agent_outcome table)
- [ ] Search module deployed? (session similarity via embeddings requires entity_embedding)
- [ ] Which tables from: agent_session ✓, agent_interaction ✓, learned_strategy, user_preference, discovered_pattern

---

## 2. Structural Checklist

**All Memory tables:**
- [ ] `GENERATED ALWAYS AS IDENTITY` surrogate key
- [ ] `scope_level VARCHAR(20) NOT NULL` present
- [ ] `scope_identifier VARCHAR(100)` present
- [ ] `COMMENT ON TABLE` — describes purpose and "big questions, small answers" principle
- [ ] `COMMENT ON COLUMN` — all columns have meaningful metadata

**Boolean flag verification (corrected conventions applied):**
- [ ] `learned_strategy.is_active` — `BYTEINT NOT NULL DEFAULT 1` (not `CHAR(1) DEFAULT 'Y'`)
- [ ] `learned_strategy.is_validated` — `BYTEINT NOT NULL DEFAULT 0` (not `CHAR(1) DEFAULT 'N'`)
- [ ] `user_preference.is_active` — `BYTEINT NOT NULL DEFAULT 1`
- [ ] `discovered_pattern.is_active` — `BYTEINT NOT NULL DEFAULT 1`
- [ ] `discovered_pattern.is_validated` — `BYTEINT NOT NULL DEFAULT 0` (renamed from `validated`)
- [ ] No `validated CHAR(1)` column anywhere — must be `is_validated BYTEINT`
- [ ] No `DEFAULT 'Y'` or `DEFAULT 'N'` on any boolean column

**agent_interaction specific:**
- [ ] `referenced_tables` is VARCHAR (comma-separated) — not a FK, not JSON
- [ ] `query_result_count` is an INTEGER count — confirmed no individual key storage
- [ ] No columns storing result record keys or customer/entity data

**discovered_pattern specific:**
- [ ] `sample_size` and `occurrences` are aggregate integers — not individual keys
- [ ] `involved_tables` is VARCHAR (comma-separated) — not individual entity keys

---

## 3. Anti-Pattern Audit — Instance Data Check

```sql
-- Check Memory tables for suspicious column names suggesting instance data storage
SELECT TableName, ColumnName, ColumnType
FROM DBC.ColumnsV
WHERE DatabaseName = '{MemoryDatabase}'
  AND ColumnName NOT IN (
    -- Expected Memory column names
    'session_key', 'session_id', 'agent_id', 'user_id', 'user_group',
    'session_start_dts', 'session_end_dts', 'session_status',
    'session_goal', 'session_context_json',
    'interaction_key', 'interaction_seq', 'interaction_type', 'interaction_dts',
    'user_input', 'agent_response', 'action_taken',
    'referenced_tables', 'sql_executed', 'query_result_count', 'execution_time_ms',
    'outcome_status', 'user_feedback',
    'strategy_key', 'strategy_name', 'strategy_description', 'strategy_category',
    'applies_to_scenario', 'strategy_pattern', 'strategy_metadata_json',
    'discovered_dts', 'discovered_by_agent', 'times_used', 'success_rate',
    'preference_key', 'preference_category', 'preference_name',
    'preference_value', 'preference_value_json',
    'applies_to_entity', 'learned_from_interactions', 'confidence', 'last_used_dts',
    'pattern_key', 'pattern_name', 'pattern_description', 'pattern_type',
    'pattern_definition_json', 'sample_size', 'occurrences',
    'confidence_score', 'statistical_significance', 'involved_tables',
    'scope_level', 'scope_identifier',
    'is_active', 'is_validated', 'validation_dts',
    'created_at', 'updated_at'
  )
ORDER BY TableName, ColumnName;
-- Any unexpected column warrants review — may indicate instance data leaking in

-- Scale check: Memory tables should have thousands of rows, not millions
SELECT TableName,
    (CurrentPerm / 1024 / 1024) AS size_mb
FROM DBC.TableSizeV
WHERE DatabaseName = '{MemoryDatabase}'
ORDER BY CurrentPerm DESC;
-- Flag any table approaching millions of rows for review
```

---

## 4. Privacy Validation Tests

```sql
-- Test 1: Every row in every Memory table has a scope_level
SELECT 'agent_session' AS table_name, COUNT(*) AS missing_scope
FROM Memory.agent_session WHERE scope_level IS NULL
UNION ALL
SELECT 'agent_interaction', COUNT(*)
FROM Memory.agent_interaction WHERE scope_level IS NULL
UNION ALL
SELECT 'learned_strategy', COUNT(*)
FROM Memory.learned_strategy WHERE scope_level IS NULL
UNION ALL
SELECT 'user_preference', COUNT(*)
FROM Memory.user_preference WHERE scope_level IS NULL
UNION ALL
SELECT 'discovered_pattern', COUNT(*)
FROM Memory.discovered_pattern WHERE scope_level IS NULL;
-- Expected: 0 for every table

-- Test 2: User scope queries return only that user's data
SELECT COUNT(*) AS accessible_strategies
FROM Memory.learned_strategy
WHERE scope_level = 'USER'
  AND scope_identifier = '{test_user_id}';
-- Expected: only strategies scoped to that specific user

-- Test 3: Organisation scope is accessible without scope_identifier filter
SELECT COUNT(*) AS org_strategies
FROM Memory.v_learned_strategies_org;
-- Expected: rows returned (if strategies exist at ORGANIZATION scope)

-- Test 4: No PII stored in user_id columns
-- Verify user_id values are identifiers, not names/emails
SELECT DISTINCT
    LEFT(user_id, 20) AS user_id_sample
FROM Memory.user_preference
SAMPLE 20;
-- Review: should be IDs like 'EMP001', 'john.doe', not 'John Doe' or phone numbers
```

---

## 5. Boolean Convention Validation

```sql
-- Verify all boolean columns are BYTEINT (no CHAR(1) booleans remaining)
SELECT TableName, ColumnName, ColumnType, DefaultValue
FROM DBC.ColumnsV
WHERE DatabaseName = '{MemoryDatabase}'
  AND ColumnName IN ('is_active', 'is_validated')
ORDER BY TableName, ColumnName;
-- Expected: ColumnType = 'I1' (Teradata BYTEINT), DefaultValue = '1' or '0'
-- Flag any ColumnType = 'CV' (VARCHAR/CHAR) — indicates old CHAR(1) pattern

-- Verify no old column name 'validated' exists (should be 'is_validated')
SELECT TableName, ColumnName
FROM DBC.ColumnsV
WHERE DatabaseName = '{MemoryDatabase}'
  AND ColumnName = 'validated';
-- Expected: 0 rows — if any found, rename to is_validated
```

---

## 6. Functional Validation Tests

```sql
-- Test 1: Session lifecycle works correctly
INSERT INTO Memory.agent_session
(session_id, agent_id, user_id, session_start_dts, session_status,
 session_goal, scope_level, scope_identifier)
VALUES
('TEST-SESSION-001', 'test_agent', 'test_user',
 CURRENT_TIMESTAMP(6), 'ACTIVE',
 'Validation test session', 'USER', 'test_user');

-- Verify active session appears in view
SELECT COUNT(*) FROM Memory.v_active_sessions
WHERE session_id = 'TEST-SESSION-001';
-- Expected: 1

-- Close the session
UPDATE Memory.agent_session
SET session_end_dts = CURRENT_TIMESTAMP(6), session_status = 'COMPLETED'
WHERE session_id = 'TEST-SESSION-001';

-- Verify session no longer active
SELECT COUNT(*) FROM Memory.v_active_sessions
WHERE session_id = 'TEST-SESSION-001';
-- Expected: 0

-- Test 2: learned_strategy boolean defaults correct
INSERT INTO Memory.learned_strategy
(strategy_name, strategy_category, strategy_pattern,
 discovered_dts, discovered_by_agent, success_rate, scope_level)
VALUES
('TEST_STRATEGY', 'QUERY_OPTIMIZATION', 'SELECT ... WHERE is_current = 1',
 CURRENT_TIMESTAMP(6), 'test_agent', 0.95, 'ORGANIZATION');

SELECT is_active, is_validated FROM Memory.learned_strategy
WHERE strategy_name = 'TEST_STRATEGY';
-- Expected: is_active = 1, is_validated = 0

-- Test 3: Strategy does NOT appear in org view until validated
SELECT COUNT(*) FROM Memory.v_learned_strategies_org
WHERE strategy_name = 'TEST_STRATEGY';
-- Expected: 0 (is_validated = 0 excludes it)

-- Validate and check it appears
UPDATE Memory.learned_strategy
SET is_validated = 1, validation_dts = CURRENT_TIMESTAMP(6)
WHERE strategy_name = 'TEST_STRATEGY';

SELECT COUNT(*) FROM Memory.v_learned_strategies_org
WHERE strategy_name = 'TEST_STRATEGY';
-- Expected: 1

-- Cleanup
DELETE FROM Memory.learned_strategy WHERE strategy_name = 'TEST_STRATEGY';
DELETE FROM Memory.agent_session WHERE session_id = 'TEST-SESSION-001';
```

---

## 7. Semantic Module Registration Checklist

```sql
-- entity_metadata: one row per Memory table
INSERT INTO Semantic.entity_metadata
(entity_name, entity_description, module_name, database_name,
 table_name, view_name, surrogate_key_column, natural_key_column,
 temporal_pattern, current_flag_column, deleted_flag_column, is_active)
VALUES
('AgentSession',
 'Agent session state — tracks active and completed sessions for continuity across interactions',
 'Memory', '{MemoryDatabase}',
 'agent_session', 'v_active_sessions', 'session_key', 'session_id',
 'NONE', NULL, NULL, 'Y'),

('AgentInteraction',
 'Interaction log — records agent actions, table references, and outcomes at table level (never instance keys)',
 'Memory', '{MemoryDatabase}',
 'agent_interaction', 'v_interactions_summary', 'interaction_key', NULL,
 'NONE', NULL, NULL, 'Y'),

('LearnedStrategy',
 'Successful patterns learned by agents — is_active=1 in use, is_validated=1 for verified strategies',
 'Memory', '{MemoryDatabase}',
 'learned_strategy', 'v_learned_strategies_org', 'strategy_key', NULL,
 'NONE', 'is_active', NULL, 'Y'),

('UserPreference',
 'User preferences learned from interactions — scope_level=USER for personal, TEAM for shared',
 'Memory', '{MemoryDatabase}',
 'user_preference', NULL, 'preference_key', NULL,
 'NONE', 'is_active', NULL, 'Y'),

('DiscoveredPattern',
 'Analytical patterns discovered by agents — statistical summaries only, never instance data',
 'Memory', '{MemoryDatabase}',
 'discovered_pattern', NULL, 'pattern_key', NULL,
 'NONE', 'is_active', NULL, 'Y');

-- data_product_map: update Memory module status
UPDATE Semantic.data_product_map
SET deployment_status = 'DEPLOYED',
    deployed_dts      = CURRENT_TIMESTAMP(6),
    primary_tables    = 'agent_session, agent_interaction, learned_strategy',
    primary_views     = 'v_active_sessions, v_interactions_summary, v_learned_strategies_org',
    updated_at        = CURRENT_TIMESTAMP(6)
WHERE module_name = 'Memory' AND is_active = 'Y';
```

---

## 8. Documentation Capture Checklist

- [ ] `dp_documentation.Module_Registry` INSERT generated (module_name = 'MEMORY', version 1.0.0)
- [ ] `dp_documentation.Design_Decision` INSERTs generated (minimum 3 decisions)
- [ ] Decision IDs follow `DD-MEMORY-{NNN}` convention
- [ ] Every decision has all ADR fields: context, alternatives_considered, rationale, consequences
- [ ] Decision categories from standard set: ARCHITECTURE | SCHEMA | NAMING | PERFORMANCE | SECURITY | INTEGRATION | OPERATIONAL
- [ ] All decisions: `decision_status = 'ACCEPTED'`, `is_current = 1`
- [ ] `dp_documentation.Change_Log` INSERT generated (CL-MEMORY-001, INITIAL_RELEASE)
- [ ] Change log includes `migration_steps` and `rollback_steps`
- [ ] `dp_documentation.Business_Glossary` INSERTs generated (minimum 3 terms)
- [ ] `dp_documentation.Query_Cookbook` INSERTs generated (minimum 1 recipe)
- [ ] All INSERTs use consistent `data_product` value
- [ ] All temporal fields: `valid_from = CURRENT_DATE`, `valid_to = DATE '9999-12-31'`
- [ ] `source_module` and `module_version` populated on every row

---

## 9. Final Pre-Delivery Checklist

- [ ] All boolean flags are `BYTEINT NOT NULL DEFAULT 1/0` — no `CHAR(1)` booleans
- [ ] `discovered_pattern.is_validated` confirmed (not `validated CHAR(1)`)
- [ ] Every table has `scope_level` + `scope_identifier` — privacy test passes (0 NULL rows)
- [ ] `referenced_tables` and `involved_tables` store table names only — confirmed no instance keys
- [ ] Anti-pattern audit run — no unexpected columns in Memory tables
- [ ] Scale check — no Memory table approaching millions of rows
- [ ] All three standard views created and tested
- [ ] `v_learned_strategies_org` correctly excludes `is_validated = 0` strategies
- [ ] Observability feed query tested against real `agent_outcome` data (or stubbed)
- [ ] Retention policy documented and purge process designed per table
- [ ] Semantic module updated: entity_metadata × 5 tables, data_product_map updated
- [ ] User IDs confirmed as identifiers not PII (privacy validation test reviewed)
- [ ] Documentation capture complete — all dp_documentation INSERTs generated
