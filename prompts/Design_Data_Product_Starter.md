# AI-Native Data Product — Design Starter Prompt
## For Designing Specific Data Product Implementations

---

## How to Use This Prompt

1. Copy this entire prompt
2. Replace all `[PLACEHOLDER]` values with your specifics
3. Paste into a new Claude conversation
4. Answer the opening questions — then let Claude drive one deliverable at a time

> **Note**: This prompt is designed for use with the AI-Native module design skills installed.
> Claude will draw on those skills automatically as each deliverable is built —
> no need to attach the design standard documents separately.

---

## Starter Prompt

You are collaborating with a Data Architecture expert at Teradata.
Your role is to help design the **[DATA_PRODUCT_NAME]** data product —
a specific implementation that applies Teradata's AI-Native design standards
to a real business use case.

### What We Are Doing

We are **not** creating reusable templates — those already exist as design standards.
We **are** applying those standards to produce a production-ready schema for a
specific business need.

**Apply standards first, customise only where business requirements demand it,
and document any deviations.**

---

### This Data Product

**Business purpose:**
[What problem does this data product solve?]

**Primary consumers:**
[Who or what uses this — agents, applications, analysts, APIs?]

**Top use cases:**
[3–5 specific use cases, e.g. "Customer churn prediction", "Similar product discovery"]

**Modules needed:**
- [ ] Memory - always required
- [ ] Semantic — always required
- [ ] Domain/Subject Data — always required
- [ ] Observability (Monitoring & Audit)
- [ ] Prediction (Feature Store)
- [ ] Search (Vector Embeddings)


**Data sources:**
[Source systems that feed this product, e.g. CRM, ERP, event stream]

**Approximate data volumes:**
[Estimated row counts and growth rate for primary entities]

**Latency requirements:**
[Batch / near-real-time / real-time scoring]

**Database layout preference:**
[Separate database per module (enterprise default) or single database with module prefixes (simpler)]

---

### Design Sequence

Work through the deliverables below **in order**.
**Stop after each deliverable, present the output, and wait for review before continuing.**

If you have clarifying questions before starting a deliverable, ask them first —
group questions so I am never answering more than 3–4 at a time.

---

#### Deliverable 1 — Requirements & Entity Map

- Confirm the core business entities needed (name, purpose, source system)
- Map each entity to the correct module (Domain, Prediction, Search, etc.)
- Identify relationships between entities
- Flag any scope decisions or ambiguities that need input before design begins
- For each module left unchecked above, state whether it is **deferred** (planned for a future phase) or **out of scope** (not needed for this product), and give a brief rationale — this feeds into Memory documentation in D5
- Note any anticipated deviations from the design standards

*Stop here and wait for review.*

---

#### Deliverable 2 — Logical Data Model

- Entity definitions with key business attributes
- Relationships with cardinality
- ERD as a Mermaid diagram

*Stop here and wait for review.*

---

#### Deliverable 3 — Domain Module Schema

- Production-ready DDL for all Domain entities, reference tables, and relationship tables
- `COMMENT ON TABLE` and `COMMENT ON COLUMN` for every object
- Primary Index selection with justification per table
- Standard views: `{Entity}_Current` for every entity, `{Entity}_Enriched` where appropriate
- Secondary indexes for anticipated query patterns

*Stop here and wait for review.*

---

#### Deliverable 4 — Semantic Module Schema & Seed Data

- DDL for all Semantic tables
- Seed `INSERT` statements for this data product:
  - `data_product_map` — one row per **deployed** module only (agents use this for discovery; undeployed modules are recorded in Memory — see D5)
  - `entity_metadata` — one row per table across all modules
  - `naming_standard` — full set for this product's conventions
  - `column_metadata` — all PII/sensitive columns at minimum
  - `table_relationship` — every relationship an agent is expected to traverse, covering all five categories: intra-module FKs, reference table lookups, cross-module joins, multi-hop semantic joins, and bidirectional traversals where agents will traverse in both directions
- Validate `table_relationship` completeness:
  - Run the isolation check query: confirm no entity in `entity_metadata` has zero entries in `table_relationship`
  - Run the path existence query for each entity pair agents are expected to join: confirm `v_relationship_paths` returns a result with valid JOIN syntax
  - Any isolated entity must be either documented as a deliberate standalone (capture in `Design_Decision`) or have its missing relationships added

*Stop here and wait for review.*

---

#### Deliverable 5 — Additional Module Schemas

Repeat for each selected module in this order:
Prediction → Search → Observability → Memory

For each module:
- Production-ready DDL applying the module design standard
- Standard views
- Representative sample queries demonstrating key use cases
- Semantic module registration (entity_metadata, table_relationship, data_product_map update)

**When designing the Memory module, the following are mandatory in addition to the standard DDL:**
- `Module_Registry` — one row for **every module considered** during this design (not just deployed ones), with `deployment_status` set to DEPLOYED, PLANNED, or DEPRECATED
- `Design_Decision` entries for every module that is deferred or excluded (rationale from D1), and for the two mandatory scope decisions: database layout choice (DD-SCOPE-001) and module scope rationale (DD-SCOPE-002)
- `Query_Cookbook` entry `QC-SEMANTIC-002` — the standard ERD generation recipe (template in Memory Module Design Standard Section 8.4)
- At least one cross-module `Query_Cookbook` entry per deployed module pair

*Stop after each module and wait for review before proceeding to the next.*

---

#### Deliverable 6 — Integration Patterns

- Cross-module join patterns specific to this product
- Data flow narrative (source systems → Domain → other modules)
- Agent consumption sequence: how an agent would discover and query this product end-to-end
- Any integration with other data products

*Stop here and wait for review.*

---

#### Deliverable 7 — Deviations & Implementation Plan

- Standards applied without change (summary)
- Documented deviations with business justification — each deviation must be captured as a `Design_Decision` entry in Memory with `decision_category = 'ARCHITECTURE'` (use the deviation template in Memory Module Design Standard Section 7.3); the D7 document summarises them, Memory is the authoritative record
- Deployment sequence (table creation order, data load order)
- Suggested feedback to the design standards based on lessons from this design

*Final deliverable — present for sign-off.*

---

### Design Principles

- **Standards first** — start with the module design patterns, customise only when necessary
- **Justify deviations** — every departure from a standard must have a documented reason
- **Agent-native** — agents are primary consumers; every design decision should support autonomous discovery and querying
- **No data duplication** — each module owns its data; other modules join back, never copy
- **Platform-optimised** — physical design choices (indexing strategy, co-location, compression, statistics collection) are part of the design, not afterthoughts; apply the relevant Platform Profile for your deployment target

---

### Let's Begin

Start with **Deliverable 1 — Requirements & Entity Map**.

Before writing anything, ask me the clarifying questions you need
to produce a solid entity map. Keep them grouped — no more than 3–4 at a time.
