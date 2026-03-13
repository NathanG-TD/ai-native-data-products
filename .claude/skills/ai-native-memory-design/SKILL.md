---
name: ai-native-memory-design
description: >
  Design the Memory module for an AI-Native Data Product on Teradata.
  Produces DDL for agent state and learning tables (agent_session, agent_interaction,
  learned_strategy, user_preference, discovered_pattern), standard views, session continuity
  patterns, privacy scoping, and Semantic module registration. Use this skill whenever the
  user asks to design or build a Memory module, implement agent session continuity, store
  learned strategies or query patterns, capture user preferences, enable cross-agent learning,
  or implement the closed-loop feedback pattern from Observability into Memory.
---

# AI-Native Memory Module Design Skill

## Two Core Principles (Never Violate)

**Principle 1: Store agent metadata — never business data.**

| Store in Memory ✅ | Never store ❌ |
|--------------------|---------------|
| "Queried Party_H, returned 125,000 results" | The 125,000 party_key values |
| SQL that was executed | Query result dataset |
| `referenced_tables = 'Domain.Party_H'` (table name) | Customer names, attributes |
| "Pattern: filter is_current first = 3× faster" | Individual record details |

**Scale check**: Memory should have thousands to tens of thousands of rows — not millions. If it's growing into millions, instance data is leaking in.

**Principle 2: Privacy scope is mandatory on every row.**

Every Memory table record must carry:
```
scope_level      VARCHAR(20) NOT NULL  -- USER | TEAM | ORGANIZATION | AGENT
scope_identifier VARCHAR(100)          -- user_id | team_id | org_id | agent_id
```

---

## Role in the Architecture

Memory is the **last module built** — Phase 3, after Observability. It is fed by Observability and feeds forward into future agent behaviour.

```
Observability  →  Memory (learns from outcomes)
Memory         →  Agents (provides context, strategies, preferences)
Memory         +  Search (find similar historical sessions via embedding)
Memory         +  Semantic (apply learned business rules)
```

---

## Design Workflow

### Step 1 — Gather Requirements

Confirm or ask for:
- Agent types in use (analytics_agent, customer_service_agent, …)
- Session patterns expected (query sessions, analysis workflows, …)
- Which learning categories are needed (query optimisation, feature selection, …)
- Scope levels required (USER only? TEAM? ORGANIZATION?)
- Privacy and data minimisation constraints (what user identifiers are acceptable?)
- Retention policies per table
- Whether Search module is deployed (enables session similarity via embeddings)

### Step 2 — Key Decisions

**Which tables to implement:**

| Table | Include When |
|-------|-------------|
| `agent_session` | Always — foundation for all other tables |
| `agent_interaction` | Always — core interaction log |
| `learned_strategy` | Agents should share successful query/approach patterns |
| `user_preference` | Agent should personalise behaviour per user |
| `discovered_pattern` | Agents perform analytical discovery and should remember findings |

**VARCHAR vs JSON storage:**

| Use VARCHAR (comma-separated) | Use JSON |
|------------------------------|----------|
| Table/entity references: `referenced_tables`, `involved_tables` | Complex nested structures: `session_context_json`, `pattern_definition_json` |
| Simple lists needing SQL `LIKE` filtering | Flexible/varying schemas |
| When SQL-level search is sufficient | Documents agent retrieves and processes in application layer |

**Retention policy** (define before go-live):
- `agent_session`: 90 days (purge completed/abandoned)
- `agent_interaction`: 1 year (audit trail)
- `learned_strategy`: 2 years (validated patterns persist longer)
- `user_preference`: active user lifecycle
- `discovered_pattern`: indefinite if validated

### Step 3 — Build in This Order

1. `agent_session` — foundation; other tables FK to it
2. `agent_interaction` — interaction log linked to sessions
3. `learned_strategy` — successful patterns from Observability feed
4. `user_preference` — personalisation layer
5. `discovered_pattern` — analytical findings
6. Standard views — `v_active_sessions`, `v_interactions_summary`, `v_learned_strategies_org`

### Step 4 — Documentation Capture

Read `../ai-native-documentation-design/references/documentation-capture.md` for the full protocol, SQL templates, and ID conventions.

**Module short name:** `MEMORY` — Decision IDs: `DD-MEMORY-{NNN}`, Change log: `CL-MEMORY-{NNN}`, Cookbook: `QC-MEMORY-{NNN}`

**Typical decisions to capture:**

| Decision | Category | Affects |
|----------|----------|---------|
| Session strategy (session lifecycle, status values) | ARCHITECTURE | `agent_session` |
| Privacy scoping (which scope levels, identifier format) | SECURITY | All Memory tables |
| Tables implemented (which of the 5 tables) | ARCHITECTURE | Module scope |
| Entity reference pattern (table-level, VARCHAR) | SCHEMA | `agent_interaction`, `discovered_pattern` |
| Retention policy per table | OPERATIONAL | All tables |
| Observability feed design (frequency, filters, scope) | INTEGRATION | `learned_strategy` |

**Typical glossary terms:** agent session, agent interaction, learned strategy, user preference, discovered pattern, scope level, privacy scoping

**Typical cookbook entries:** active session retrieval, organisation-scoped strategy query, Observability-to-Memory feed query

**Required outputs:**
1. `dp_documentation.Module_Registry` INSERT (1 — module registration)
2. `dp_documentation.Design_Decision` INSERTs (minimum 3 — key architectural/schema choices)
3. `dp_documentation.Change_Log` INSERT (1 — CL-MEMORY-001, INITIAL_RELEASE v1.0.0)
4. `dp_documentation.Business_Glossary` INSERTs (minimum 3 — domain terms introduced)
5. `dp_documentation.Query_Cookbook` INSERTs (minimum 1 — key query patterns)

### Step 5 — Validate

Run the scale check and privacy tests in `references/checklists.md`.

---

## Boolean Conventions

All boolean flags follow Domain module standards throughout Memory:

```
is_active      BYTEINT NOT NULL DEFAULT 1   -- 1 = active, 0 = deprecated
is_validated   BYTEINT NOT NULL DEFAULT 0   -- 1 = validated, 0 = not yet validated
```

Filters: `= 1` and `= 0` — never `= 'Y'` or `= 'N'`.

---

## Entity Reference Pattern (Table Level)

Memory references **tables by name**, never by instance key:

```sql
-- referenced_tables and involved_tables: comma-separated qualified names
'Domain.Party_H, Prediction.customer_features'
'Domain.Party_H'
'Domain.Product_H, Search.product_embedding'

-- SQL filtering (TESTED ✅ — no JSON parsing needed)
WHERE referenced_tables LIKE '%Domain.Party_H%'     -- specific table
WHERE referenced_tables LIKE '%Domain.%'            -- any Domain table
WHERE referenced_tables LIKE '%Party_H%'            -- table in any database
WHERE referenced_tables LIKE '%Party_H%'
  AND referenced_tables LIKE '%customer_features%'  -- multiple tables
```

---

## Quick Reference

**Scope query patterns:**
```sql
WHERE scope_level = 'USER'         AND scope_identifier = '{user_id}'
WHERE scope_level = 'TEAM'         AND scope_identifier = '{team_id}'
WHERE scope_level = 'ORGANIZATION'                              -- all users
WHERE scope_level = 'AGENT'        AND scope_identifier = '{agent_id}'
```

**Standard views:**
```
v_active_sessions          -- Sessions with session_status = 'ACTIVE'
v_interactions_summary     -- Interaction log with VARCHAR table references
v_learned_strategies_org   -- ORGANIZATION scope, is_active = 1, is_validated = 1
```

**Observability → Memory feed trigger:**
Scheduled job reads `Observability.agent_outcome` (last 7 days, SUCCESS, execution_time_ms < 1000, COUNT ≥ 10) → writes to `Memory.learned_strategy` at ORGANIZATION scope.

---

## Reference Files

| File | Read When |
|------|-----------|
| `references/memory-tables.md` | Writing DDL for all five tables and standard views |
| `references/memory-queries.md` | Session continuity, learning queries, Observability feed, agent discovery |
| `references/checklists.md` | Final review, scale check, privacy validation, Semantic registration |
| `../ai-native-documentation-design/references/documentation-capture.md` | Documentation Capture step — SQL templates and ID conventions |
