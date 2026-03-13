# Memory Tables & Views Reference
## DDL Templates for AI-Native Memory Module

Replace `Memory` with your actual database name (e.g., `Customer360_Memory`).

**Key conventions:**
- All boolean flags: `BYTEINT NOT NULL DEFAULT value` with `is_` prefix
- `referenced_tables` and `involved_tables`: VARCHAR comma-separated qualified names (`database.table`)
- All tables have `scope_level` + `scope_identifier` — mandatory privacy scoping
- Scale target: thousands to tens of thousands of rows, never millions

---

## 1. agent_session — Session State

Foundation table. All other Memory tables relate back to sessions via `session_key`.

```sql
CREATE TABLE Memory.agent_session (
    session_key             INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for session record',

    -- Session identification
    session_id              VARCHAR(100) NOT NULL
        COMMENT 'Business session identifier — unique ID for tracking session across systems',
    agent_id                VARCHAR(100) NOT NULL
        COMMENT 'Agent identifier — which agent instance is handling this session',
    user_id                 VARCHAR(100)
        COMMENT 'User identifier — which user is interacting with agent (store ID only, not PII)',

    -- Session lifecycle
    session_start_dts       TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When agent session began',
    session_end_dts         TIMESTAMP(6) WITH TIME ZONE
        COMMENT 'When session completed or was abandoned — NULL for active sessions',
    session_status          VARCHAR(20)
        COMMENT 'Session state: ACTIVE | COMPLETED | ABANDONED',

    -- Session content
    session_goal            VARCHAR(500)
        COMMENT 'What the user is trying to accomplish in this session',
    session_context_json    JSON
        COMMENT 'Flexible session context — JSON document with session-specific metadata. Agent retrieves and processes in application layer.',

    -- Privacy scope (REQUIRED on every Memory record)
    scope_level             VARCHAR(20) NOT NULL
        COMMENT 'Privacy scope: USER (private) | TEAM (team-shared) | ORGANIZATION (company-wide) | AGENT (agent-specific)',
    scope_identifier        VARCHAR(100)
        COMMENT 'Scope identifier: user_id for USER, team_id for TEAM, org_id for ORGANIZATION, agent_id for AGENT',

    -- Audit
    created_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When session record was created'
)
PRIMARY INDEX (session_key);

COMMENT ON TABLE Memory.agent_session IS
'Agent session state — tracks active and historical sessions for continuity across interactions.
 session_context_json holds flexible context; agent retrieves and processes externally.
 scope_level + scope_identifier enforce privacy boundaries on every record.
 Active sessions: session_status = ''ACTIVE'' AND session_end_dts IS NULL.';
```

---

## 2. agent_interaction — Interaction Log

Append-only log of what happened within each session. References tables by name, never by instance key.

```sql
CREATE TABLE Memory.agent_interaction (
    interaction_key         INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for interaction record',

    -- Session link
    session_key             INTEGER NOT NULL
        COMMENT 'FK to Memory.agent_session.session_key — links interaction to parent session',

    -- Interaction details
    interaction_seq         INTEGER NOT NULL
        COMMENT 'Sequence number within session — orders interactions chronologically',
    interaction_type        VARCHAR(50)
        COMMENT 'Interaction category: QUERY | ACTION | DECISION | EXPLANATION',
    interaction_dts         TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When this interaction occurred',

    -- What happened
    user_input              VARCHAR(4000)
        COMMENT 'User question or request — what the user asked the agent to do',
    agent_response          VARCHAR(4000)
        COMMENT 'Agent response summary — brief description of what was provided or done',
    action_taken            VARCHAR(500)
        COMMENT 'Specific action executed by agent',

    -- Entity references — TABLE LEVEL ONLY (comma-separated qualified names)
    referenced_tables       VARCHAR(1000)
        COMMENT 'Tables involved: comma-separated database.table names — TABLE LEVEL, never individual record keys. Format: Domain.Party_H, Prediction.customer_features. Filter with LIKE.',

    -- Query metadata
    sql_executed            VARCHAR(4000)
        COMMENT 'SQL query executed by agent — actual statement, if applicable',
    query_result_count      INTEGER
        COMMENT 'Aggregate count of records returned — NEVER individual record keys',
    execution_time_ms       INTEGER
        COMMENT 'Query execution time in milliseconds',

    -- Outcome
    outcome_status          VARCHAR(20)
        COMMENT 'Interaction result: SUCCESS | PARTIAL | FAILED',
    user_feedback           VARCHAR(20)
        COMMENT 'Human feedback: POSITIVE | NEUTRAL | NEGATIVE | NULL (no feedback) — drives agent learning',

    -- Privacy scope (REQUIRED)
    scope_level             VARCHAR(20) NOT NULL
        COMMENT 'Privacy scope: USER | TEAM | ORGANIZATION | AGENT',
    scope_identifier        VARCHAR(100)
        COMMENT 'Scope identifier matching scope_level',

    -- Audit
    created_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When interaction record was created'
)
PRIMARY INDEX (interaction_key);

COMMENT ON TABLE Memory.agent_interaction IS
'Agent interaction log — records what agent did, which tables were involved, and outcomes.
 referenced_tables stores table names only (database.table format), NEVER instance keys.
 query_result_count is an aggregate count only — never store individual result record keys.
 Filter referenced_tables with LIKE: WHERE referenced_tables LIKE ''%Domain.Party_H%''.';
```

---

## 3. learned_strategy — Successful Patterns

Stores reusable query patterns and approaches discovered through successful agent interactions. Fed primarily from the Observability module.

```sql
CREATE TABLE Memory.learned_strategy (
    strategy_key            INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for learned strategy record',

    -- Strategy identity
    strategy_name           VARCHAR(100) NOT NULL
        COMMENT 'Descriptive strategy identifier',
    strategy_description    VARCHAR(1000)
        COMMENT 'Explains what this strategy does and when to apply it',

    -- Strategy context
    strategy_category       VARCHAR(50)
        COMMENT 'Category: QUERY_OPTIMIZATION | FEATURE_SELECTION | ERROR_HANDLING | DATA_QUALITY | ANALYSIS_PATTERN',
    applies_to_scenario     VARCHAR(500)
        COMMENT 'Conditions or context where this strategy is effective',

    -- Strategy definition
    strategy_pattern        VARCHAR(4000)
        COMMENT 'Implementation: SQL pattern, logic description, or approach details',
    strategy_metadata_json  JSON
        COMMENT 'Complex strategy details as JSON — agent retrieves and processes in application layer',

    -- Learning provenance
    discovered_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When strategy was discovered',
    discovered_by_agent     VARCHAR(100)
        COMMENT 'Agent ID that learned this pattern',
    times_used              INTEGER NOT NULL DEFAULT 0
        COMMENT 'Usage count — how many times strategy has been applied',
    success_rate            DECIMAL(5,4)
        COMMENT 'Success rate: 0.0–1.0 — percentage of applications that led to successful outcome',

    -- Privacy scope (REQUIRED)
    scope_level             VARCHAR(20) NOT NULL
        COMMENT 'Visibility: USER | TEAM | ORGANIZATION | AGENT',
    scope_identifier        VARCHAR(100)
        COMMENT 'Scope identifier matching scope_level',

    -- Status
    is_active               BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Active flag: 1 = strategy in use, 0 = deprecated or superseded',
    is_validated            BYTEINT NOT NULL DEFAULT 0
        COMMENT 'Validation flag: 1 = validated by human expert or testing, 0 = discovered but not yet validated',
    validation_dts          TIMESTAMP(6) WITH TIME ZONE
        COMMENT 'When strategy was validated — NULL until validated',

    -- Audit
    created_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When strategy record was created',
    updated_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When strategy was last updated or refined'
)
PRIMARY INDEX (strategy_key);

COMMENT ON TABLE Memory.learned_strategy IS
'Strategies learned by agents — successful patterns discovered through experience and Observability feedback.
 is_active = 1 for usable strategies; is_validated = 1 for human-verified strategies.
 ORGANIZATION scope strategies are shared across all agents.
 Feed from Observability: high-success-rate patterns from agent_outcome become strategies here.';
```

---

## 4. user_preference — Personalisation

Preferences learned per user, enabling personalised agent behaviour over time.

```sql
CREATE TABLE Memory.user_preference (
    preference_key          INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for preference record',

    -- User identification (store ID only — no PII)
    user_id                 VARCHAR(100) NOT NULL
        COMMENT 'User identifier — store user_id only, never names or emails',
    user_group              VARCHAR(100)
        COMMENT 'User group or team — for group-level preference aggregation',

    -- Preference definition
    preference_category     VARCHAR(50)
        COMMENT 'Category: REPORT_FORMAT | DATA_FILTER | AGGREGATION_LEVEL | VISUALIZATION_TYPE | ANALYSIS_DEPTH',
    preference_name         VARCHAR(100) NOT NULL
        COMMENT 'Specific preference identifier within category',
    preference_value        VARCHAR(1000)
        COMMENT 'Simple text preference value for basic preferences',
    preference_value_json   JSON
        COMMENT 'Structured preference for complex values — agent retrieves and processes in application layer',

    -- Context
    applies_to_entity       VARCHAR(100)
        COMMENT 'Table name or entity type this preference applies to',
    applies_to_scenario     VARCHAR(500)
        COMMENT 'Context or use case where this preference applies',

    -- Learning confidence
    learned_from_interactions INTEGER
        COMMENT 'Number of interactions that contributed to this preference — confidence indicator',
    confidence              DECIMAL(5,4)
        COMMENT 'Confidence in preference: 0.0–1.0, higher = stronger evidence',
    last_used_dts           TIMESTAMP(6) WITH TIME ZONE
        COMMENT 'When preference was last applied — tracks recency',

    -- Privacy scope (REQUIRED)
    scope_level             VARCHAR(20) NOT NULL DEFAULT 'USER'
        COMMENT 'Visibility: USER (default, personal) | TEAM | ORGANIZATION',

    -- Status
    is_active               BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Active flag: 1 = preference in use, 0 = deprecated or superseded',

    -- Audit
    created_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When preference was first learned',
    updated_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When preference was last updated or reinforced'
)
PRIMARY INDEX (preference_key);

COMMENT ON TABLE Memory.user_preference IS
'User preferences learned from interactions — enables personalised agent behaviour.
 Store user_id only, never PII (names, emails, demographics).
 is_active = 1 for current preferences; scope_level defaults to USER (private).
 preference_value for simple values; preference_value_json for structured preferences.';
```

---

## 5. discovered_pattern — Analytical Findings

Patterns discovered by agents through data analysis. Stores statistical summary only — never the underlying record data.

```sql
CREATE TABLE Memory.discovered_pattern (
    pattern_key             INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY
        COMMENT 'Surrogate key for discovered pattern record',

    -- Pattern identity
    pattern_name            VARCHAR(100) NOT NULL
        COMMENT 'Descriptive identifier for discovered insight',
    pattern_description     VARCHAR(1000)
        COMMENT 'Explains the pattern and its business implications',

    -- Pattern definition
    pattern_type            VARCHAR(50)
        COMMENT 'Discovery type: CORRELATION | TEMPORAL | TABLE_RELATIONSHIP | ANOMALY | SEGMENTATION',
    pattern_definition_json JSON
        COMMENT 'Pattern details, conditions, and formulas as JSON — agent retrieves and processes in application layer',

    -- Statistical support (summary statistics ONLY — never individual record keys)
    sample_size             INTEGER
        COMMENT 'Records analysed to discover pattern — evidence strength indicator. Never store the actual record keys.',
    occurrences             INTEGER
        COMMENT 'Times pattern was observed — frequency indicator',
    confidence_score        DECIMAL(5,4)
        COMMENT 'Statistical confidence: 0.0–1.0',
    statistical_significance DECIMAL(5,4)
        COMMENT 'p-value or significance score for pattern validity testing',

    -- Discovery metadata
    discovered_dts          TIMESTAMP(6) WITH TIME ZONE NOT NULL
        COMMENT 'When pattern was discovered',
    discovered_by_agent     VARCHAR(100)
        COMMENT 'Agent ID that performed the discovery analysis',

    -- Entity references — TABLE LEVEL ONLY
    involved_tables         VARCHAR(1000)
        COMMENT 'Tables involved: comma-separated database.table names — TABLE LEVEL, never instance keys. Filter with LIKE.',

    -- Privacy scope (REQUIRED)
    scope_level             VARCHAR(20) NOT NULL
        COMMENT 'Visibility: USER | TEAM | ORGANIZATION | AGENT',
    scope_identifier        VARCHAR(100)
        COMMENT 'Scope identifier matching scope_level',

    -- Status
    is_validated            BYTEINT NOT NULL DEFAULT 0
        COMMENT 'Validation flag: 1 = validated by human expert or testing, 0 = discovered but unvalidated',
    is_active               BYTEINT NOT NULL DEFAULT 1
        COMMENT 'Active flag: 1 = pattern current, 0 = deprecated or invalidated',

    -- Audit
    created_at              TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
        COMMENT 'When pattern record was created'
)
PRIMARY INDEX (pattern_key);

COMMENT ON TABLE Memory.discovered_pattern IS
'Patterns discovered by agents through data analysis — statistical summaries only, never individual records.
 sample_size and occurrences are aggregate counts; involved_tables stores table names not record keys.
 is_validated = 1 for patterns confirmed by human experts or testing.
 pattern_definition_json holds complex pattern details for agent application-layer processing.';
```

---

## 6. Standard Views

### `v_active_sessions`

```sql
CREATE VIEW Memory.v_active_sessions AS
SELECT
    session_key,
    session_id,
    agent_id,
    user_id,
    session_start_dts,
    session_goal,
    scope_level,
    scope_identifier
FROM Memory.agent_session
WHERE session_status = 'ACTIVE'
  AND session_end_dts IS NULL;

COMMENT ON VIEW Memory.v_active_sessions IS
'Currently active agent sessions — filters to ACTIVE status with no end timestamp.
 Use to resume session context or detect orphaned sessions.';
```

### `v_interactions_summary`

```sql
CREATE VIEW Memory.v_interactions_summary AS
SELECT
    ai.interaction_key,
    ai.session_key,
    ai.interaction_seq,
    ai.interaction_type,
    ai.user_input,
    ai.action_taken,
    ai.sql_executed,
    ai.query_result_count,
    ai.execution_time_ms,
    ai.outcome_status,
    ai.user_feedback,
    ai.referenced_tables,       -- VARCHAR — filter directly with LIKE
    ai.scope_level,
    ai.scope_identifier,
    ai.interaction_dts
FROM Memory.agent_interaction ai;

COMMENT ON VIEW Memory.v_interactions_summary IS
'Agent interaction log — direct access to interaction metadata and table references.
 Filter referenced_tables with LIKE: WHERE referenced_tables LIKE ''%Domain.Party_H%''.
 No JSON parsing needed — referenced_tables is plain VARCHAR comma-separated.';
```

### `v_learned_strategies_org`

```sql
CREATE VIEW Memory.v_learned_strategies_org AS
SELECT
    strategy_key,
    strategy_name,
    strategy_description,
    strategy_category,
    applies_to_scenario,
    strategy_pattern,
    success_rate,
    times_used,
    discovered_by_agent,
    discovered_dts,
    validation_dts
FROM Memory.learned_strategy
WHERE scope_level  = 'ORGANIZATION'
  AND is_active    = 1
  AND is_validated = 1
ORDER BY success_rate DESC, times_used DESC;

COMMENT ON VIEW Memory.v_learned_strategies_org IS
'Organisation-wide validated strategies — active, validated, ORGANIZATION-scoped strategies only.
 Ordered by success_rate then usage frequency for agent strategy selection.
 Use as the primary source for agents selecting approaches to apply.';
```
