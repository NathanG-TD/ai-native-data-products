# AI-Native Module Skill Conversion Prompt
## Converting Design Standard Documents into Claude Skills

---

## Purpose

This prompt converts an AI-Native Data Product module design standard document
into a Claude skill using the progressive disclosure pattern.

The **design standard document is the master source of truth**.
The skill is a compressed, optimised view of that document —
structured for efficient agent consumption, not independent of it.

When the design document changes, the skill must be updated to match.
When there is a conflict between skill content and design document content,
the design document wins.

---

## How to Use This Prompt

1. Start a new Claude conversation
2. Attach the **Master Design Standard** document
3. Attach the **module design standard document** you are converting
4. Attach the **Advocated Data Management Standards** document (if converting Domain or a module that references it)
5. Paste this prompt

If updating an existing skill rather than creating a new one, also provide the current `.skill` file.

---

## Conversion Prompt

You are converting an AI-Native Data Product module design standard document
into a Claude skill. The skill creator tool is available in your skills library —
use it to package the final output.

The design standard document is the **master source of truth**.
The skill is a compressed, structured view of that knowledge — not a copy.
Every design decision and DDL pattern in the skill must be traceable to the source document.

---

### Step 1 — Read and Analyse

Read all attached documents before writing anything.

From the module design standard, extract and note:

- **Module purpose** — one sentence, the primary thing this module enables
- **Core principle** — the one rule that must never be violated (e.g. "store keys only, not content")
- **Tables to design** — list every table with its purpose
- **Design decisions** — choices the designer must make before writing DDL (storage pattern, temporal strategy, indexing, etc.)
- **Integration points** — how this module connects to Domain and other modules
- **Standard views** — which views are required vs optional
- **Agent discovery patterns** — how agents find and query this module
- **What the source document omits** — common operational patterns not covered (DML, refresh, Semantic registration, etc.)

Then identify any **consistency issues** before writing a single line of output:

Check every boolean-style column in the source document against these standards:
- Boolean flags must use `BYTEINT NOT NULL DEFAULT 1/0` — never `CHAR(1) DEFAULT 'Y'/'N'`
- Boolean columns must use the `is_` prefix — e.g. `is_current`, `is_active`, `is_validated`
- Filter values must be `= 1` or `= 0` — never `= 'Y'` or `= 'N'`
- Temporal tables use `PRIMARY INDEX` (NUPI) — never `UNIQUE PRIMARY INDEX`

**Stop and report any inconsistencies found before proceeding.**
List the table name, column name, current definition, and the corrected definition.
Wait for confirmation of the correction approach before continuing.

---

### Step 2 — Apply the Progressive Disclosure Pattern

Structure the skill using three loading layers:

```
Layer 1 — SKILL.md          (always loaded, target: 100–175 lines)
Layer 2 — reference files   (loaded on demand, 200–400 lines each)
Layer 3 — [future: assets]  (not used in current module skills)
```

**SKILL.md contains:**
- YAML frontmatter: `name` and `description`
- Core principle (never-violate rule)
- Role in the six-module architecture (one compact table)
- Design workflow (steps, not DDL)
- Key decision tables (what drives each design choice)
- Quick reference: naming patterns, standard filters, column conventions
- Reference file index (what each file contains, when to load it)

**SKILL.md does NOT contain:**
- Full DDL templates
- Query syntax
- INSERT/UPDATE/DELETE patterns
- Checklists
- Semantic registration SQL

**Reference files** — split content by when it is needed:

| File | Contains | Load when |
|------|----------|-----------|
| `{module}-tables.md` | DDL templates, views, anti-patterns | Writing table definitions |
| `{module}-queries.md` | Query patterns, DML, operational patterns | Writing queries or data operations |
| `checklists.md` | Design review, validation tests, Semantic registration | Final review before delivery |

If a module has a critical complex object (e.g. a recursive CTE, a specialised function call),
give it prominent placement in the relevant reference file with a clear "do not modify" note
if the syntax has been validated and must be preserved exactly.

---

### Step 3 — Optimise for Context Efficiency

The goal is to leave the agent maximum working memory for the actual design task.
Apply these compression techniques when moving from source document to skill:

**Replace verbose prose with decision tables:**
```
Instead of three paragraphs explaining when to use wide vs tall format —
| Condition | Pattern |
|-----------|---------|
| Features always accessed together, fixed set | Wide format |
| Features accessed individually, sparse | Tall format |
```

**Replace explanatory examples with parameterised templates:**
```
Instead of a Customer-specific example showing party_key —
CREATE TABLE {Schema}.{entity}_embedding (
    {entity}_key BIGINT NOT NULL ...
```

**Consolidate repeated patterns:**
If the source document shows the same temporal column set five times across
five different tables, define it once in the reference file with a note
"apply this pattern to all temporal tables in this module."

**Preserve exactly (do not compress):**
- Validated SQL syntax for complex objects (recursive CTEs, TD_VectorDistance calls)
- Teradata-specific syntax that has known failure modes if altered
- COMMENT ON TABLE / COLUMN text — these are agent-readable metadata, not documentation

**Add what is typically missing from source documents:**
Source design documents describe structure but often omit operational patterns.
Always add to the queries reference file:
- The expire-current → insert-new DML pattern for temporal tables
- A refresh/regeneration pattern (how to update records when source data changes)
- An agent discovery query sequence (how an agent first encounters this module)
- A staleness detection query (how to find records that need refresh)

Always add to the checklists reference file:
- Full Semantic module registration SQL (entity_metadata, column_metadata,
  table_relationship, data_product_map UPDATE) as ready-to-run INSERT statements
- A functional validation test block (can be run to confirm the design is correct)
- An anti-pattern audit query (catches common mistakes for this specific module)

---

### Step 4 — Apply Consistent Conventions Throughout

Apply these conventions in every file you produce.
These are non-negotiable — they exist so agents learn patterns once and
apply them everywhere.

**Boolean columns:**
```sql
-- Always BYTEINT, always is_ prefix, always NOT NULL
is_current      BYTEINT NOT NULL DEFAULT 1
is_active       BYTEINT NOT NULL DEFAULT 1
is_validated    BYTEINT NOT NULL DEFAULT 0
is_deleted      BYTEINT NOT NULL DEFAULT 0
is_threshold_met BYTEINT NOT NULL DEFAULT 0
is_sla_met      BYTEINT NOT NULL DEFAULT 0

-- Filters: always integers, never strings
WHERE is_current = 1
WHERE is_active = 1 AND is_validated = 0
```

**Temporal tables:**
```sql
-- Always NUPI for tables with multiple versions per key
PRIMARY INDEX ({entity}_key)   -- Non-unique, allows multiple versions

-- Never UNIQUE PRIMARY INDEX on temporal tables
-- Reference tables (no versioning) may use UNIQUE PRIMARY INDEX
```

**Timestamps:**
```sql
-- Always WITH TIME ZONE, always (6) precision
TIMESTAMP(6) WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
TIMESTAMP '9999-12-31 23:59:59.999999+00:00'   -- "open" end date
```

**Table suffixes:**
```
_H       History/versioned entity table
_R       Reference/lookup table
_Current View: current active records (is_current = 1 AND is_deleted = 0)
_Enriched View: with common Domain joins
```

**Cross-module references:**
- VARCHAR comma-separated for table name lists: `'Domain.Party_H, Prediction.customer_features'`
- BIGINT foreign keys for entity references: `entity_key BIGINT` or `{entity}_key BIGINT`
- Never duplicate content from another module — join back at query time

---

### Step 5 — Write the Files

Write files in this order:

1. **SKILL.md** — workflow and decisions only, no DDL
2. **`{module}-tables.md`** — DDL templates with full COMMENT ON statements
3. **`{module}-queries.md`** — query patterns, DML operations, agent discovery
4. **`checklists.md`** — design review, validation tests, Semantic registration

After writing each file, verify:
- No `CHAR(1)` boolean columns
- No `= 'Y'` or `= 'N'` filter values
- No content duplication from other modules (check for columns that belong in Domain)
- SKILL.md is under 175 lines
- Each reference file is under 400 lines

If a reference file approaches 400 lines, consider splitting it.
Common split: separate the DDL-heavy tables file from the query-heavy patterns file.

---

### Step 6 — Package and Validate

Use the skill creator packaging tool:

```bash
python -m scripts.package_skill /home/claude/{skill-directory-name} /home/claude
```

The validator must return `✅ Skill is valid!` before proceeding.

If validation fails, fix the reported issues, do not skip validation.

Copy the packaged `.skill` file to `/mnt/user-data/outputs/` and present it
to the user for download.

---

### Step 7 — Report

After packaging, provide a brief summary covering:

1. **Skill structure** — file names and line counts
2. **Context efficiency** — SKILL.md lines, typical session context (SKILL.md + 1–2 files)
3. **Consistency corrections** — any boolean or naming fixes applied (with old → new)
4. **Source document gaps filled** — operational patterns added that were absent from the source
5. **Source document update required** — if corrections were applied, list the exact changes
   the designer should make to keep the design document in sync with the skill

---

## Reference: Progressive Disclosure Targets

| File | Target | Hard limit |
|------|--------|-----------|
| SKILL.md | 100–150 lines | 175 lines |
| Any single reference file | 200–350 lines | 400 lines |
| Total (all files combined) | < 1,000 lines | 1,200 lines |
| Typical session context | 300–500 lines | — |

## Reference: Skill Directory Structure

```
ai-native-{module}-design/
├── SKILL.md
└── references/
    ├── {module}-tables.md
    ├── {module}-queries.md
    └── checklists.md
```

## Reference: Modules and Implementation Order

| Order | Module | Skill name |
|-------|--------|-----------|
| 1 | Domain/Subject Data | `ai-native-domain-design` |
| 2 | Semantic | `ai-native-semantic-design` |
| 3 | Prediction | `ai-native-prediction-design` |
| 3 | Search | `ai-native-search-design` |
| 4 | Observability | `ai-native-observability-design` |
| 4 | Memory | `ai-native-memory-design` |

Domain and Semantic are always built first (Phase 1).
Prediction and Search are Phase 2.
Observability and Memory are Phase 3.

---

## When Updating an Existing Skill

If the design document has been updated and the skill needs to reflect those changes:

1. Attach the updated design document and the current `.skill` file
2. Run Step 1 (Read and Analyse) comparing old and new document
3. List every change in the design document and its impact on the skill files
4. Confirm the change list with the user before editing
5. Apply changes to the relevant reference files using `str_replace` — do not rewrite whole files
6. Re-run Step 4 (convention check) and Step 6 (package and validate)
7. Report what changed and confirm the source document is now the master again

The skill name and directory name must not change between versions.
