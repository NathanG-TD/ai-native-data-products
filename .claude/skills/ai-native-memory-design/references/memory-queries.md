# Memory Query Patterns Reference
## Session Continuity, Learning Retrieval, Observability Feed, Agent Discovery

---

## 1. Session Continuity Patterns

### Resume an Existing Session

```sql
-- Retrieve session context to resume where agent left off
SELECT
    s.session_id,
    s.session_goal,
    s.session_context_json,     -- Parse in application layer for full context
    s.scope_level,
    s.scope_identifier
FROM Memory.agent_session s
WHERE s.session_id = '{session_id}'
  AND s.session_status = 'ACTIVE';

-- Get last N interactions from the session for context window
SELECT
    interaction_seq,
    interaction_type,
    user_input,
    agent_response,
    action_taken,
    referenced_tables,
    outcome_status,
    sql_executed
FROM Memory.agent_interaction
WHERE session_key = {session_key}
ORDER BY interaction_seq DESC
QUALIFY ROW_NUMBER() OVER (ORDER BY interaction_seq DESC) <= 10;
```

### Close a Session

```sql
UPDATE Memory.agent_session
SET session_end_dts = CURRENT_TIMESTAMP(6),
    session_status  = 'COMPLETED'   -- or 'ABANDONED'
WHERE session_key = {session_key}
  AND session_status = 'ACTIVE';
```

---

## 2. Strategy Retrieval for Agent Decision-Making

### Get Applicable Strategies for a Scenario

```sql
-- Retrieve organisation-wide validated strategies for agent to apply
SELECT
    strategy_name,
    strategy_category,
    strategy_pattern,
    success_rate,
    times_used,
    applies_to_scenario
FROM Memory.v_learned_strategies_org
WHERE strategy_category = '{QUERY_OPTIMIZATION|FEATURE_SELECTION|ERROR_HANDLING}'
  AND success_rate >= 0.85
ORDER BY success_rate DESC, times_used DESC;

-- Strategies referencing a specific table (any scope visible to user)
SELECT
    strategy_name,
    strategy_pattern,
    success_rate,
    scope_level
FROM Memory.learned_strategy
WHERE (
    strategy_pattern LIKE '%{TableName}%'
    OR applies_to_scenario LIKE '%{TableName}%'
)
AND is_active    = 1
AND is_validated = 1
AND (
    scope_level = 'ORGANIZATION'
    OR (scope_level = 'TEAM' AND scope_identifier = '{team_id}')
    OR (scope_level = 'USER' AND scope_identifier = '{user_id}')
)
ORDER BY success_rate DESC;
```

### Update Strategy Usage on Application

```sql
-- Increment usage count and update success rate after applying a strategy
UPDATE Memory.learned_strategy
SET times_used  = times_used + 1,
    -- Recalculate success rate incrementally
    success_rate = ((success_rate * times_used) + {outcome_score}) / (times_used + 1),
    updated_at   = CURRENT_TIMESTAMP(6)
WHERE strategy_key = {strategy_key};
-- outcome_score: 1.0 for SUCCESS, 0.5 for PARTIAL, 0.0 for FAILED
```

---

## 3. User Preference Retrieval

```sql
-- Get all active preferences for a user
SELECT
    preference_category,
    preference_name,
    preference_value,
    preference_value_json,    -- Parse complex prefs in application layer
    applies_to_entity,
    confidence
FROM Memory.user_preference
WHERE user_id     = '{user_id}'
  AND scope_level = 'USER'
  AND is_active   = 1
ORDER BY preference_category, confidence DESC;

-- Get team-shared preferences a user can inherit
SELECT
    preference_category,
    preference_name,
    preference_value,
    confidence
FROM Memory.user_preference
WHERE scope_level      = 'TEAM'
  AND scope_identifier = '{team_id}'
  AND is_active        = 1
ORDER BY preference_category, confidence DESC;

-- Update/reinforce a preference based on new interaction
UPDATE Memory.user_preference
SET learned_from_interactions = learned_from_interactions + 1,
    confidence  = LEAST(1.0, confidence + 0.05),  -- Increment with ceiling
    last_used_dts = CURRENT_TIMESTAMP(6),
    updated_at    = CURRENT_TIMESTAMP(6)
WHERE user_id           = '{user_id}'
  AND preference_name   = '{preference_name}'
  AND is_active         = 1;
```

---

## 4. Pattern Discovery Queries

```sql
-- Find validated patterns involving a specific table
SELECT
    pattern_name,
    pattern_description,
    pattern_type,
    sample_size,
    occurrences,
    confidence_score,
    involved_tables
FROM Memory.discovered_pattern
WHERE involved_tables LIKE '%{TableName}%'
  AND is_validated = 1
  AND is_active    = 1
ORDER BY confidence_score DESC;

-- Organisation-wide patterns available to all agents
SELECT
    pattern_name,
    pattern_type,
    pattern_definition_json,    -- Agent processes in application layer
    confidence_score,
    sample_size,
    involved_tables
FROM Memory.discovered_pattern
WHERE scope_level    = 'ORGANIZATION'
  AND is_validated   = 1
  AND is_active      = 1
  AND confidence_score >= 0.8
ORDER BY confidence_score DESC;

-- Unvalidated patterns awaiting review
SELECT
    pattern_name,
    pattern_description,
    discovered_by_agent,
    confidence_score,
    sample_size,
    discovered_dts,
    involved_tables
FROM Memory.discovered_pattern
WHERE is_validated = 0
  AND is_active    = 1
ORDER BY confidence_score DESC, discovered_dts DESC;
```

---

## 5. Interaction History Analysis

```sql
-- What interactions involved a specific table?
SELECT
    interaction_seq,
    interaction_type,
    user_input,
    action_taken,
    outcome_status,
    sql_executed,
    query_result_count,
    execution_time_ms
FROM Memory.v_interactions_summary
WHERE referenced_tables LIKE '%{database}.{TableName}%'
  AND scope_level = 'ORGANIZATION'  -- or USER/TEAM with scope_identifier
ORDER BY interaction_dts DESC;

-- Which SQL patterns were most successful for a given table?
SELECT
    sql_executed,
    COUNT(*) AS usage_cnt,
    AVG(execution_time_ms) AS avg_ms,
    SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS success_cnt,
    CAST(SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS DECIMAL(10,4))
        / COUNT(*) AS success_rate
FROM Memory.agent_interaction
WHERE referenced_tables LIKE '%{TableName}%'
  AND sql_executed IS NOT NULL
GROUP BY sql_executed
HAVING COUNT(*) >= 3
ORDER BY success_rate DESC, avg_ms ASC;

-- Agent performance over time
SELECT
    CAST(interaction_dts AS DATE) AS interaction_date,
    interaction_type,
    COUNT(*) AS total_cnt,
    SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS success_cnt,
    AVG(execution_time_ms) AS avg_ms
FROM Memory.agent_interaction
WHERE agent_id   = '{agent_id}'
  AND interaction_dts >= CURRENT_TIMESTAMP(6) - INTERVAL '30' DAY
GROUP BY CAST(interaction_dts AS DATE), interaction_type
ORDER BY interaction_date DESC;
```

---

## 6. Observability → Memory Feed (Closed-Loop Learning)

Scheduled job — run daily or weekly. Reads successful outcomes from Observability and promotes them to organisation-wide strategies.

```sql
-- Insert new learned strategies from recent Observability agent outcomes
INSERT INTO Memory.learned_strategy (
    strategy_name,
    strategy_description,
    strategy_category,
    applies_to_scenario,
    strategy_pattern,
    discovered_dts,
    discovered_by_agent,
    times_used,
    success_rate,
    scope_level,
    is_active,
    is_validated
)
SELECT
    'AutoLearned_' || action_type || '_' ||
        CAST(CURRENT_DATE AS VARCHAR(10))       AS strategy_name,
    'Auto-discovered from Observability: ' || action_type ||
        ' actions on ' || MIN(tables_accessed) ||
        ' averaged ' || CAST(CAST(AVG(execution_time_ms) AS INTEGER) AS VARCHAR(10)) ||
        'ms with ' || CAST(CAST(
            SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
        AS INTEGER) AS VARCHAR(10)) || '% success rate'
                                                AS strategy_description,
    'QUERY_OPTIMIZATION'                        AS strategy_category,
    'Action type: ' || action_type ||
        ' involving: ' || MIN(tables_accessed)  AS applies_to_scenario,
    'Action type: ' || action_type ||
        ' — see tables_accessed in Observability.agent_outcome for detail'
                                                AS strategy_pattern,
    CURRENT_TIMESTAMP(6)                        AS discovered_dts,
    'observability_feed'                        AS discovered_by_agent,
    COUNT(*)                                    AS times_used,
    CAST(SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS DECIMAL(10,4))
        / COUNT(*)                              AS success_rate,
    'ORGANIZATION'                              AS scope_level,
    1                                           AS is_active,
    0                                           AS is_validated  -- Requires human validation
FROM Observability.agent_outcome
WHERE action_dts        >= CURRENT_TIMESTAMP(6) - INTERVAL '7' DAY
  AND outcome_status     = 'SUCCESS'
  AND execution_time_ms  < 1000
GROUP BY action_type
HAVING COUNT(*)         >= 10    -- Minimum sample for statistical reliability
   AND CAST(SUM(CASE WHEN outcome_status = 'SUCCESS' THEN 1 ELSE 0 END) AS DECIMAL(10,4))
       / COUNT(*)        >= 0.9; -- 90%+ success rate threshold
```

---

## 7. Agent Session Recording Patterns

### Record a New Session

```sql
INSERT INTO Memory.agent_session
(session_id, agent_id, user_id,
 session_start_dts, session_status,
 session_goal, session_context_json,
 scope_level, scope_identifier)
VALUES
('{session_id}', '{agent_id}', '{user_id}',
 CURRENT_TIMESTAMP(6), 'ACTIVE',
 '{session_goal}',
 '{"initial_context": "{context}", "data_product": "{product_name}"}',
 'USER', '{user_id}');
```

### Log an Interaction

```sql
INSERT INTO Memory.agent_interaction
(session_key, interaction_seq, interaction_type, interaction_dts,
 user_input, agent_response, action_taken,
 referenced_tables,
 sql_executed, query_result_count, execution_time_ms,
 outcome_status, user_feedback,
 scope_level, scope_identifier)
VALUES
({session_key},
 (SELECT COALESCE(MAX(interaction_seq), 0) + 1
  FROM Memory.agent_interaction WHERE session_key = {session_key}),
 '{QUERY|ACTION|DECISION|EXPLANATION}',
 CURRENT_TIMESTAMP(6),
 '{user_question}',
 '{brief_response_summary}',
 '{action_description}',
 '{database}.{Table1}, {database}.{Table2}',   -- Table names only, no instance keys
 '{sql_text_or_NULL}',
 {result_count_or_NULL},
 {execution_ms_or_NULL},
 '{SUCCESS|PARTIAL|FAILED}',
 NULL,                                          -- Updated when user provides feedback
 '{scope_level}', '{scope_identifier}');
```

### Update Feedback After User Response

```sql
UPDATE Memory.agent_interaction
SET user_feedback = '{POSITIVE|NEUTRAL|NEGATIVE|CORRECTION}'
WHERE interaction_key = {interaction_key};
```
