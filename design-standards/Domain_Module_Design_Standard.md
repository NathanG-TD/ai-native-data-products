# Domain Module Design Standard
## AI-Native Data Product Architecture - Version 2.0

---

## Document Control

| Attribute | Value |
|-----------|-------|
| **Version** | 2.0 |
| **Status** | STANDARD |
| **Last Updated** | 2025-02-09 |
| **Owner** | Nathan Green, Worldwide Data Architecture Team, Teradata |
| **Scope** | Domain/Subject Data Module Structure |
| **Type** | Design Standard (Structural Requirements) |
| **Companion** | Advocated Data Management Standards (Implementation Guidance) |

---

## Table of Contents

1. [AI-Native Domain Module Overview](#1-ai-native-domain-module-overview)
2. [Module Scope and Boundaries](#2-module-scope-and-boundaries)
3. [Standard Naming Conventions](#3-standard-naming-conventions)
4. [Entity Structure Patterns](#4-entity-structure-patterns)
5. [Cross-Module Integration Standards](#5-cross-module-integration-standards)
6. [Agent Discoverability Requirements](#6-agent-discoverability-requirements)
7. [Physical Design Considerations](#7-physical-design-considerations)
8. [Designer Responsibilities](#8-designer-responsibilities)

---

## 1. AI-Native Domain Module Overview

### 1.1 What Makes a Domain Module AI-Native?

The Domain Module is the **authoritative source of truth** for core business entities. What makes it AI-Native is not a specific temporal pattern or column set, but rather:

| AI-Native Characteristic | Purpose |
|-------------------------|---------|
| **Temporal Integrity** | Enables point-in-time state reconstruction for ML features |
| **Agent Discoverability** | Consistent patterns agents can learn once and reuse |
| **Cross-Module Integration** | Standardized references other modules can rely on |
| **Complete Lineage** | Full traceability for explainability |
| **Metadata Richness** | Self-documenting for autonomous agent operation |

### 1.2 Domain Module Role in AI-Native Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AI-Native Data Product                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐                                        │
│  │ Domain/Subject  │ ← Authoritative entities & relationships│
│  │     Module      │                                        │
│  └────────┬────────┘                                        │
│           │                                                  │
│           │ Join-back foundation for:                       │
│           │                                                  │
│  ┌────────┴────────┬──────────┬──────────┬──────────┐      │
│  │                 │          │          │          │      │
│  ▼                 ▼          ▼          ▼          ▼      │
│ Search         Prediction  Observability Semantic  Memory  │
│ (Vectors)      (Features)  (Events)     (Rules)    (State) │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

All other modules reference Domain entities using standardized patterns.

### 1.3 Design Flexibility

**This standard defines STRUCTURE, not specific implementations.**

You have flexibility in:
- Temporal tracking approach (bi-temporal, Type 2 SCD, event sourcing)
- Column sets (minimal core vs extended metadata)
- Key strategy (surrogate vs natural keys)
- Delete approach (soft vs hard)
- Storage optimization (normalization vs denormalization)

**Requirement**: Whatever approach you choose must support the AI-Native characteristics in Section 1.1.

**Guidance**: See **"Advocated Data Management Standards"** document for Teradata's recommended implementations.

---

## 2. Module Scope and Boundaries

### 2.1 What Belongs in Domain Module

**IN SCOPE:**
- Core business entities (Party, Product, Location, Agreement, etc.)
- Entity attributes and their history
- Relationships between entities
- Reference data and taxonomies
- **Domain-related events and transactions** (customer transactions, product usage, agreement changes)
- **Domain-specific measurements** (account balances, transaction amounts, usage metrics)

**Key Principle**: If it represents the **business domain** (what the business does), it belongs in Domain.

### 2.2 What Belongs in Other Modules

| Module | Contains | Example |
|--------|----------|---------|
| **Prediction** | Derived ML features | Customer churn probability, product affinity scores |
| **Search** | Vector embeddings | Semantic representations for similarity search |
| **Observability** | Data product operational data | Model predictions, agent actions, quality assessments |
| **Semantic** | Ontologies and rules | Relationship definitions, business rules |
| **Memory** | Agent conversational state | Dialog history, learned preferences |

**Key Principle**: If it's about the **AI data product's operation**, it belongs in other modules.

### 2.3 Observability vs Domain Events

| Event Type | Module | Example |
|------------|--------|---------|
| **Business domain event** | Domain | Customer purchases product, Account balance changes |
| **Data product operational event** | Observability | Model makes prediction, Data quality check runs |
| **Change tracking event** | Observability | Entity attribute updated, Record deleted |

---

## 3. Agent Discoverability Design Principles

### 3.1 Why Agent Discoverability Matters

AI agents need to work autonomously with data without human instruction. This requires:

| Capability | Enables Agent To... |
|------------|---------------------|
| **Self-discovery** | Find entities and understand their purpose |
| **Pattern recognition** | Generalize from one entity to others |
| **Relationship navigation** | Traverse entity relationships automatically |
| **Query generation** | Construct valid queries programmatically |
| **Context understanding** | Interpret data meaning and usage |

**Key Principle**: Consistency enables pattern learning. Agents learn once and apply everywhere.

### 3.2 Critical Design Techniques for Agent Discoverability

#### Technique 1: Rich Metadata (REQUIRED)

**Every table and column MUST have descriptive metadata:**

```sql
CREATE TABLE Party_H (
    party_key BIGINT NOT NULL
        COMMENT 'Unique system identifier for Party entity, never reused, used for joins',
    
    party_id VARCHAR(50) NOT NULL
        COMMENT 'Natural business identifier from source system, used for user queries and external references',
    
    legal_name VARCHAR(200)
        COMMENT 'Legal name of party as registered with government authorities or business registries',
    
    tax_identifier VARCHAR(50)
        COMMENT 'Tax identification number (SSN for individuals, EIN for organizations), encrypted at rest',
    
    -- All columns must have meaningful COMMENT
);
```

**Agent Usage:**
```sql
-- Agent discovers what's available
SELECT TableName, CommentString
FROM DBC.TablesV
WHERE DatabaseName = 'CustomerDomain'
ORDER BY TableName;

-- Agent understands column meanings
SELECT ColumnName, ColumnType, CommentString
FROM DBC.ColumnsV
WHERE DatabaseName = 'CustomerDomain'
  AND TableName = 'Party_H'
ORDER BY ColumnId;
```

**Quality Criteria for Metadata:**
- ✅ Describes business meaning, not technical structure
- ✅ Includes context (what, why, when to use)
- ✅ Notes constraints or special handling
- ✅ References source if from external system
- ❌ Avoid: "Party key" (redundant with column name)
- ❌ Avoid: Technical jargon without explanation

#### Technique 2: Consistent Patterns (REQUIRED)

**Agents learn patterns through consistency. Design for pattern recognition:**

**Pattern Example: Entity Identification**
```sql
-- If one entity has these columns...
CREATE TABLE Party_H (
    party_key BIGINT,      -- Surrogate key
    party_id VARCHAR(50),  -- Natural key
    ...
);

-- Then ALL entities should follow same pattern
CREATE TABLE Product_H (
    product_key BIGINT,      -- Surrogate key (same pattern)
    product_id VARCHAR(50),  -- Natural key (same pattern)
    ...
);

CREATE TABLE Agreement_H (
    agreement_key BIGINT,      -- Surrogate key (same pattern)
    agreement_id VARCHAR(50),  -- Natural key (same pattern)
    ...
);
```

**Agent Learning**: "Every entity has `{entity}_key` for internal references and `{entity}_id` for business references"

**Pattern Example: Current State**
```sql
-- If one entity uses this pattern...
SELECT * FROM Party_H WHERE is_current = 1 AND is_deleted = 0;

-- Then ALL entities should use same pattern
SELECT * FROM Product_H WHERE is_current = 1 AND is_deleted = 0;
SELECT * FROM Agreement_H WHERE is_current = 1 AND is_deleted = 0;
```

**Agent Learning**: "To get current active records, always filter on `is_current = 1 AND is_deleted = 0`"

#### Technique 3: Naming Consistency (RECOMMENDED)

**While enterprise standards may vary, consistency WITHIN your data product is critical:**

**Example 1: Temporal Columns**
```sql
-- Good: Consistent temporal pattern across entities
CREATE TABLE Party_H (
    valid_from_dts TIMESTAMP(6) WITH TIME ZONE,
    valid_to_dts TIMESTAMP(6) WITH TIME ZONE,
    ...
);

CREATE TABLE Product_H (
    valid_from_dts TIMESTAMP(6) WITH TIME ZONE,  -- Same names
    valid_to_dts TIMESTAMP(6) WITH TIME ZONE,    -- Same names
    ...
);

-- Avoid: Inconsistent naming confuses agents
CREATE TABLE Agreement_H (
    start_timestamp TIMESTAMP(6) WITH TIME ZONE,  -- Different names
    end_timestamp TIMESTAMP(6) WITH TIME ZONE,    -- Different names
    ...
);
```

**Example 2: Boolean Flags**
```sql
-- Good: Consistent boolean naming
is_current, is_deleted, is_active, is_verified

-- Avoid: Mixed boolean naming
is_current, deleted_flag, active_ind, verified_yn
```

**Example 3: Type Indicators**
```sql
-- Good: Consistent type/code columns
party_type_code, product_type_code, agreement_type_code

-- Avoid: Inconsistent type columns
party_type, product_category, agreement_kind
```

**Capture in Semantic Module**: Document your naming standards in the Semantic module so agents can reference them.

#### Technique 4: Self-Describing Relationships (REQUIRED)

**Foreign keys should be descriptive, not generic:**

```sql
-- Good: Descriptive foreign keys
CREATE TABLE PartyRole_H (
    party_role_key BIGINT,
    party_key BIGINT,              -- Clear: references Party
    role_type_code VARCHAR(20),
    context_agreement_key BIGINT,  -- Clear: references Agreement in specific context
    ...
);

-- Avoid: Generic foreign keys
CREATE TABLE PartyRole_H (
    party_role_key BIGINT,
    fk1 BIGINT,     -- Unclear: what does this reference?
    fk2 BIGINT,     -- Unclear: what does this reference?
    ...
);
```

**Agent Learning**: "Column names ending in `_key` are foreign keys, the prefix tells me what entity they reference"

#### Technique 5: Standard Views for Common Patterns (REQUIRED)

**Create predictable views that agents can discover:**

```sql
-- Standard view pattern 1: Current active records
CREATE VIEW Party_Current AS
SELECT * FROM Party_H
WHERE is_current = 1 AND is_deleted = 0;

CREATE VIEW Product_Current AS
SELECT * FROM Product_H
WHERE is_current = 1 AND is_deleted = 0;

-- Agent learns: "{Entity}_Current views always give me active records"

-- Standard view pattern 2: Enriched with common joins
CREATE VIEW Party_Enriched AS
SELECT p.*, [common join data]
FROM Party_Current p
LEFT JOIN [common related data];

-- Agent learns: "{Entity}_Enriched views give me frequently needed related data"
```

#### Technique 6: Table/View Naming Signals (RECOMMENDED)

**Use suffixes to signal table purpose to agents:**

```sql
-- Suffix signals for agents
Party_H          -- "_H" = History table (temporal, versioned)
CountryCode_R    -- "_R" = Reference data (lookup/taxonomy)
PartyProduct_H   -- Entity1+Entity2 = Relationship table
Party_Current    -- No suffix = View (simplified access)
Party_Enriched   -- No suffix = View (with joins)
```

**Capture in Semantic Module**: Document what suffixes mean in your Semantic module.

#### Technique 7: Metadata-Driven Discovery (RECOMMENDED)

**Store business rules and patterns in queryable metadata:**

```sql
-- Example: Semantic module table documenting entity patterns
CREATE TABLE Semantic.EntityMetadata (
    entity_name VARCHAR(100),
    entity_purpose VARCHAR(500),
    primary_table_name VARCHAR(100),
    current_view_name VARCHAR(100),
    surrogate_key_column VARCHAR(100),
    natural_key_column VARCHAR(100),
    current_flag_column VARCHAR(100),
    deleted_flag_column VARCHAR(100),
    source_system VARCHAR(100),
    ...
);

-- Agent queries to understand entity structure
SELECT * FROM Semantic.EntityMetadata WHERE entity_name = 'Party';
```

**Agent Learning**: "I can query metadata to understand any entity's structure without hardcoding"

### 3.3 Anti-Patterns That Break Agent Discoverability

| Anti-Pattern | Why It's Bad | Better Approach |
|-------------|--------------|-----------------|
| **Inconsistent naming** | Agent can't generalize patterns | Choose convention, apply everywhere |
| **Missing metadata** | Agent can't understand purpose | Add meaningful COMMENT to all objects |
| **Cryptic abbreviations** | Agent can't interpret meaning | Use full words or document in Semantic |
| **Hidden relationships** | Agent can't navigate | Use descriptive FK names |
| **Undocumented views** | Agent doesn't know what view does | Add COMMENT explaining view purpose |
| **Mixed patterns** | Agent must special-case each entity | Standardize patterns across entities |

### 3.4 Leveraging Enterprise Standards

**Your organization likely has:**
- Enterprise naming conventions
- Standard abbreviations dictionary
- Industry data model standards (FIBO, HL7, etc.)
- Database naming guidelines

**Best Practice**: 
1. **Adopt** your enterprise standards for naming
2. **Document** those standards in the Semantic module
3. **Apply consistently** across all entities
4. **Provide metadata** so agents can reference your standards

**Example Semantic Module Entry:**

```sql
CREATE TABLE Semantic.NamingStandards (
    standard_type VARCHAR(50),      -- 'ABBREVIATION', 'SUFFIX', 'PATTERN'
    standard_value VARCHAR(100),    -- 'dts', '_H', '{entity}_key'
    meaning VARCHAR(500),           -- 'datetime stamp', 'history table', ...
    usage_guidance VARCHAR(1000),   -- When and how to apply
    examples VARCHAR(1000)          -- Concrete examples
);

-- Agent queries naming standards
SELECT * FROM Semantic.NamingStandards 
WHERE standard_type = 'SUFFIX';
```

### 3.5 Design Checklist for Agent Discoverability

**Before finalizing entity design:**

- [ ] All tables have meaningful COMMENT describing purpose
- [ ] All columns have meaningful COMMENT describing meaning
- [ ] Patterns are consistent across all entities (keys, flags, temporal)
- [ ] Foreign keys are descriptive (not generic fk1, fk2, etc.)
- [ ] Standard views created ({Entity}_Current at minimum)
- [ ] Table/view naming signals purpose (if using suffixes)
- [ ] Enterprise naming standards applied consistently
- [ ] Naming standards documented in Semantic module (if deviating from defaults)
- [ ] Agents can discover entities via metadata queries
- [ ] Agents can understand structure without human explanation

**Test**: Can an agent that's never seen your entities:
1. Discover what entities exist?
2. Understand what each entity represents?
3. Find current active records?
4. Navigate relationships?
5. Generate valid queries?

If yes to all → Agent-discoverable ✅

---

## 4. Entity Structure Patterns

### 4.1 Core Entity Pattern

**Every Domain entity table MUST have:**

```sql
CREATE TABLE {EntityName}_H (
    -- Identity (Required)
    {entity}_key        BIGINT NOT NULL,
    {entity}_id         VARCHAR(50) NOT NULL,
    
    -- Temporal tracking (Required - implementation flexible)
    -- Designer chooses: bi-temporal, Type 2 SCD, or other
    -- Must support point-in-time state reconstruction
    -- See "Advocated Standards" for recommended approach
    
    -- Current state indicator (Required for query optimization)
    is_current          BYTEINT NOT NULL DEFAULT 1,
    
    -- Soft delete indicator (Required for audit trail)
    is_deleted          BYTEINT NOT NULL DEFAULT 0,
    
    -- Entity-specific attributes (Designer supplied)
    -- Based on enterprise data model or industry standard
    
    PRIMARY INDEX ({entity}_key)
)
-- NOTE: For temporal tables, use PRIMARY INDEX (not UNIQUE PRIMARY INDEX)
-- because multiple versions of same entity_key will exist.
-- Designer defines appropriate index based on temporal strategy chosen.
-- See Advocated Data Management Standards for detailed guidance.

-- Table comment (Required)
COMMENT ON TABLE {EntityName}_H IS 
'{Entity} history table with temporal tracking - stores all versions of {entity} over time for point-in-time reconstruction';

-- Column comments (Required for all columns)
COMMENT ON COLUMN {EntityName}_H.{entity}_key IS 
'Surrogate key - system generated unique identifier, never reused, used for all internal joins and foreign key references';

COMMENT ON COLUMN {EntityName}_H.{entity}_id IS 
'Natural business identifier from source system - used for user queries, reports, and external system integration';

COMMENT ON COLUMN {EntityName}_H.is_current IS 
'Current version indicator - 1 = current active version, 0 = historical superseded version - used for query optimization';

COMMENT ON COLUMN {EntityName}_H.is_deleted IS 
'Soft delete indicator - 1 = logically deleted but retained for audit, 0 = active record - enables audit trail without physical deletion';

-- Add comments for temporal columns (if using bi-temporal)
-- COMMENT ON COLUMN {EntityName}_H.valid_from_dts IS 'When this version became true in the real world';
-- COMMENT ON COLUMN {EntityName}_H.valid_to_dts IS 'When this version stopped being true in the real world';
-- COMMENT ON COLUMN {EntityName}_H.transaction_from_dts IS 'When this version was inserted into the database';
-- COMMENT ON COLUMN {EntityName}_H.transaction_to_dts IS 'When this version was superseded in the database';

-- Add comments for entity-specific attributes (Designer supplies)
-- COMMENT ON COLUMN {EntityName}_H.{attribute_name} IS '{Business meaning and purpose of this attribute}';
```

**Flexibility**: The specific temporal columns, audit columns, and metadata columns are designer choices.

**Requirement**: Must enable:
- Point-in-time state reconstruction
- Current state queries (`is_current = 1`)
- Exclude deleted records (`is_deleted = 0`)
- Agent discovery via metadata

### 4.2 Reference Data Pattern

```sql
CREATE TABLE {ReferenceName}_R (
    {reference}_key     BIGINT NOT NULL,
    {reference}_code    VARCHAR(20) NOT NULL,
    short_description   VARCHAR(100) NOT NULL,
    long_description    VARCHAR(500),
    
    -- Temporal (simpler than entity tables)
    effective_date      DATE NOT NULL,
    expiration_date     DATE NOT NULL DEFAULT DATE '9999-12-31',
    is_current          BYTEINT NOT NULL DEFAULT 1,
    
    -- Optional: Hierarchy
    parent_{reference}_key BIGINT,
    sort_order          INTEGER,
    
    PRIMARY INDEX ({reference}_key)
)
-- For reference data, can use UNIQUE PRIMARY INDEX on (code, effective_date)
-- since reference data typically doesn't version like entity tables:
UNIQUE PRIMARY INDEX ({reference}_code, effective_date);

-- Table comment (Required)
COMMENT ON TABLE {ReferenceName}_R IS 
'{Reference} reference data table - controlled vocabulary and lookup values with temporal validity periods';

-- Column comments (Required for all columns)
COMMENT ON COLUMN {ReferenceName}_R.{reference}_key IS 
'Surrogate key for reference data entry';

COMMENT ON COLUMN {ReferenceName}_R.{reference}_code IS 
'Reference code - short identifier used in domain tables, must be unique within effective period';

COMMENT ON COLUMN {ReferenceName}_R.short_description IS 
'Brief description of reference value - displayed in UI dropdowns and reports';

COMMENT ON COLUMN {ReferenceName}_R.long_description IS 
'Detailed description of reference value - provides complete definition and usage guidance';

COMMENT ON COLUMN {ReferenceName}_R.effective_date IS 
'Date when this reference value becomes valid for use';

COMMENT ON COLUMN {ReferenceName}_R.expiration_date IS 
'Date when this reference value expires - default 9999-12-31 for indefinite validity';

COMMENT ON COLUMN {ReferenceName}_R.is_current IS 
'Current validity indicator - 1 = currently valid, 0 = expired or future';

COMMENT ON COLUMN {ReferenceName}_R.parent_{reference}_key IS 
'Optional parent reference for hierarchical taxonomies - references {reference}_key for parent-child relationships';

COMMENT ON COLUMN {ReferenceName}_R.sort_order IS 
'Optional display sequence for ordering reference values in UI or reports';
```

### 4.3 Relationship Pattern

```sql
CREATE TABLE {Entity1}{Entity2}_H (
    {entity1}_{entity2}_key BIGINT NOT NULL,
    {entity1}_key       BIGINT NOT NULL,
    {entity2}_key       BIGINT NOT NULL,
    
    -- Temporal (same pattern as entity tables)
    -- ... temporal columns ...
    is_current          BYTEINT NOT NULL DEFAULT 1,
    is_deleted          BYTEINT NOT NULL DEFAULT 0,
    
    -- Relationship-specific attributes
    -- Designer supplies based on business requirements
    
    PRIMARY INDEX ({entity1}_key)  -- Co-locate with first entity
);

-- NOTE: For temporal relationships, use PRIMARY INDEX
-- See Advocated Data Management Standards for temporal strategy

-- Table comment (Required)
COMMENT ON TABLE {Entity1}{Entity2}_H IS 
'{Entity1} to {Entity2} relationship history - associative table with temporal tracking for many-to-many relationships';

-- Column comments (Required for all columns)
COMMENT ON COLUMN {Entity1}{Entity2}_H.{entity1}_{entity2}_key IS 
'Surrogate key for this relationship instance - unique identifier for the association';

COMMENT ON COLUMN {Entity1}{Entity2}_H.{entity1}_key IS 
'Foreign key to {Entity1}_H.{entity1}_key - identifies the first entity in the relationship';

COMMENT ON COLUMN {Entity1}{Entity2}_H.{entity2}_key IS 
'Foreign key to {Entity2}_H.{entity2}_key - identifies the second entity in the relationship';

COMMENT ON COLUMN {Entity1}{Entity2}_H.is_current IS 
'Current version indicator - 1 = current active relationship, 0 = historical superseded relationship';

COMMENT ON COLUMN {Entity1}{Entity2}_H.is_deleted IS 
'Soft delete indicator - 1 = relationship terminated but retained for audit, 0 = active relationship';

-- Add comments for temporal columns (Designer supplies based on temporal strategy)
-- Add comments for relationship-specific attributes (Designer supplies based on business requirements)
```

---

## 5. Cross-Module Integration Standards

### 5.1 Standard Foreign Key Pattern

**All other modules reference Domain entities using ONE of these patterns:**

#### Pattern A: Generic Entity Reference (Flexible)

```sql
-- Use when module references MANY different entity types
CREATE TABLE OtherModule.SomeTable (
    some_table_key      BIGINT NOT NULL,
    entity_key          BIGINT NOT NULL,      -- FK to any Domain entity
    entity_type         VARCHAR(50) NOT NULL, -- 'PARTY', 'PRODUCT', etc.
    -- ... other columns ...
);

-- Join pattern
SELECT * 
FROM OtherModule.SomeTable t
INNER JOIN Domain.Party_H p
    ON p.party_key = t.entity_key
   AND t.entity_type = 'PARTY'
WHERE p.is_current = 1 AND p.is_deleted = 0;
```

#### Pattern B: Specific Entity References (Type-Safe)

```sql
-- Use when module references FEW specific entity types
CREATE TABLE OtherModule.SomeTable (
    some_table_key      BIGINT NOT NULL,
    party_key           BIGINT,    -- NULL if not a party
    product_key         BIGINT,    -- NULL if not a product
    -- ... other columns ...
    
    FOREIGN KEY (party_key) REFERENCES Domain.Party_H(party_key),
    FOREIGN KEY (product_key) REFERENCES Domain.Product_H(product_key)
);
```

**Standard Requirement**: Choose one pattern consistently within each module.

### 5.2 Join-Back Pattern

**Standard**: All other modules join BACK to Domain for entity context.

```sql
-- Example: Prediction module joining to Domain
SELECT 
    pred.prediction_score,
    pred.model_id,
    p.party_id,          -- From Domain
    p.party_name,        -- From Domain
    p.party_type_code    -- From Domain
FROM Prediction.Prediction_H pred
INNER JOIN Domain.Party_H p
    ON p.party_key = pred.entity_key
   AND pred.entity_type = 'PARTY'
   AND p.is_current = 1
   AND p.is_deleted = 0
WHERE pred.is_current = 1;
```

**Requirement**: Domain entity context is accessed via joins, not duplicated in other modules.

### 5.3 Standard View Pattern

**Requirement**: Create standard views for common access patterns.

```sql
-- Current version view (always create)
CREATE VIEW {Entity}_Current AS
SELECT {entity}_key, {entity}_id, [business columns]
FROM {Entity}_H
WHERE is_current = 1 AND is_deleted = 0;

COMMENT ON VIEW {Entity}_Current IS 
'Current active {entity} records - filters to current non-deleted versions for simplified querying';

-- Enriched view (when joining to other modules is common)
CREATE VIEW {Entity}_Enriched AS
SELECT 
    e.*,
    [columns from other modules]
FROM {Entity}_Current e
LEFT JOIN [other modules];

COMMENT ON VIEW {Entity}_Enriched IS 
'{Entity} records enriched with data from other modules - provides complete entity context in single query';
```

**Benefit**: Simplifies agent queries while maintaining normalized storage.

**Requirement**: Add COMMENT ON VIEW statements for all views to explain their purpose and filtering logic.

---

## 6. Standard Query Patterns

### 6.1 Why Standard Query Patterns Matter

Agents need predictable query patterns they can learn once and reuse. When all entities follow the same patterns, agents can work autonomously without entity-specific programming.

### 6.2 Required Query Patterns

**Pattern 1: Get Current Active Records**

```sql
-- All entities use same pattern
SELECT * FROM {Entity}_H 
WHERE is_current = 1 AND is_deleted = 0;

-- Examples
SELECT * FROM Party_H WHERE is_current = 1 AND is_deleted = 0;
SELECT * FROM Product_H WHERE is_current = 1 AND is_deleted = 0;
SELECT * FROM Agreement_H WHERE is_current = 1 AND is_deleted = 0;
```

**Pattern 2: Get Specific Entity by Natural Key**

```sql
-- All entities use same pattern
SELECT * FROM {Entity}_H
WHERE {entity}_id = 'SPECIFIC_ID'
  AND is_current = 1 
  AND is_deleted = 0;

-- Examples
SELECT * FROM Party_H WHERE party_id = 'CUST-123' AND is_current = 1 AND is_deleted = 0;
SELECT * FROM Product_H WHERE product_id = 'SKU-456' AND is_current = 1 AND is_deleted = 0;
```

**Pattern 3: Use Standard Current View**

```sql
-- All entities provide _Current view
SELECT * FROM {Entity}_Current 
WHERE {entity}_id = 'SPECIFIC_ID';

-- Examples
SELECT * FROM Party_Current WHERE party_id = 'CUST-123';
SELECT * FROM Product_Current WHERE product_id = 'SKU-456';
```

### 6.3 Benefits of Standard Patterns

| Benefit | Description |
|---------|-------------|
| **Agent autonomy** | Agent learns pattern once, applies to all entities |
| **Reduced errors** | Standard patterns are tested and reliable |
| **Simplified training** | Show agent one example, they understand all |
| **Query generation** | Agent can construct queries programmatically |
| **Performance predictability** | Standard patterns can be optimized consistently |

### 6.4 Pattern Consistency Checklist

- [ ] All entities support `is_current = 1` filter for current version
- [ ] All entities support `is_deleted = 0` filter for active records
- [ ] All entities have `{entity}_id` for natural key queries
- [ ] All entities have `{Entity}_Current` view
- [ ] Query patterns documented in Semantic module

---

## 7. Designer Responsibilities

### 8.1 What Designers Must Supply

| Element | Source | Example |
|---------|--------|---------|
| **Entity model** | Enterprise standard or industry model | iLDM, FIBO, HL7 FHIR, custom |
| **Entity attributes** | Business requirements | Customer name, account balance |
| **Temporal strategy** | Based on use case requirements | Bi-temporal, Type 2 SCD, other |
| **Column set** | Based on audit/lineage needs | Minimal core vs extended metadata |
| **Relationships** | Business domain analysis | Party-Product associations |
| **Reference data** | Industry standards or business | Country codes, status codes |
| **Primary index** | Access pattern analysis | Surrogate key, natural key |
| **Partitioning** | Volume and access patterns | Monthly by transaction date |

### 8.2 Design Documentation Requirements

**Before implementation, document:**

- [ ] Entity purpose and business definition
- [ ] Source of entity model (iLDM, industry standard, custom)
- [ ] Entity-specific attributes with business definitions
- [ ] Natural/business key(s) identified
- [ ] Relationships to other entities mapped
- [ ] Temporal strategy chosen and justified
- [ ] Column set chosen (core vs extended) and justified
- [ ] Reference data tables identified
- [ ] Data quality rules defined
- [ ] Source system(s) identified
- [ ] Update frequency documented
- [ ] Retention policy defined
- [ ] Primary index selection justified
- [ ] Partitioning strategy justified (if applicable)
- [ ] Secondary indexes planned with rationale

### 8.3 Standard Entity Model Sources

**Designers should source entity models from:**

1. **Enterprise Data Model**
   - Integrated Logical Data Model (iLDM)
   - Common Data Model (CDM)
   - Corporate Data Dictionary

2. **Industry Standards** (by domain)
   - **Finance**: FIBO (EDM Council), BIAN, ISO 20022
   - **Healthcare**: HL7 FHIR, SNOMED CT, LOINC
   - **Retail**: GS1, ARTS
   - **Insurance**: ACORD
   - **Telecom**: TM Forum (SID)

3. **Open Standards**
   - Schema.org for common entities
   - Dublin Core for metadata
   - SKOS for taxonomies

4. **Custom Models**
   - When no standard exists
   - Document rationale and structure
   - Consider future standardization

### 8.4 Design Review Checklist

**Before finalizing design:**

- [ ] Follows standard naming conventions (Section 3)
- [ ] Has required identity columns ({entity}_key, {entity}_id)
- [ ] Has temporal tracking that supports point-in-time reconstruction
- [ ] Has is_current and is_deleted indicators
- [ ] All columns have COMMENT metadata
- [ ] Foreign key patterns follow standard (Section 5)
- [ ] Standard views defined (Section 5.3)
- [ ] Physical design choices documented and justified
- [ ] Integrates cleanly with other modules
- [ ] Agent discoverability requirements met

---

## Appendix A: Quick Reference

### Required Structural Elements

**Every Domain entity table:**
```
✅ {entity}_key          -- Surrogate key (or equivalent unique identifier)
✅ {entity}_id           -- Natural/business key  
✅ Temporal tracking     -- Designer chooses approach (must support point-in-time)
✅ is_current            -- Current version indicator
✅ is_deleted            -- Soft delete indicator
✅ COMMENT on all tables -- Purpose and usage
✅ COMMENT on all columns -- Meaning and context
✅ PRIMARY INDEX         -- Designer chooses based on access patterns
✅ UNIQUE INDEX          -- Based on temporal approach
```

### Agent Discoverability Requirements

**For agents to work autonomously:**
```
✅ Rich metadata (COMMENT) on all objects
✅ Consistent patterns across all entities
✅ Descriptive foreign key names (not generic fk1, fk2)
✅ Standard views created ({Entity}_Current minimum)
✅ Patterns documented in Semantic module
✅ Enterprise naming standards applied consistently
```

### Standard Views to Create

```
{Entity}_Current        -- Current active records (REQUIRED)
{Entity}_Enriched       -- With common joins (OPTIONAL)
{Entity}_AuditTrail     -- Complete history (OPTIONAL)
```

### Standard Query Patterns

```sql
-- Current active records (all entities)
WHERE is_current = 1 AND is_deleted = 0

-- Specific entity by natural key (all entities)
WHERE {entity}_id = 'ID' AND is_current = 1 AND is_deleted = 0

-- Using standard view (all entities)
SELECT * FROM {Entity}_Current WHERE {entity}_id = 'ID'
```

### Cross-Module Reference Patterns

```sql
-- Pattern A: Generic reference (flexible)
entity_key BIGINT, entity_type VARCHAR(50)

-- Pattern B: Specific reference (type-safe)
party_key BIGINT, product_key BIGINT
```

### Naming Consistency Guidelines

**Whatever naming conventions you use, be consistent:**
- Keys: Same pattern for all entities
- Temporal columns: Same names across entities
- Boolean flags: Same naming style
- Type/code columns: Same suffixes
- Foreign keys: Descriptive, not generic

**Document your conventions in Semantic module**

### Integration Checklist

- [ ] Required structural elements present
- [ ] Metadata (COMMENT) complete and meaningful
- [ ] Patterns consistent across all entities
- [ ] Standard views created
- [ ] Foreign key patterns clear and descriptive
- [ ] Enterprise standards followed
- [ ] Agents can discover and understand entities
- [ ] Query patterns work across all entities

---

## Document Change Log

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 2.0 | 2025-02-09 | Refactored to focus on structure, not implementation | Nathan Green, Worldwide Data Architecture Team, Teradata |
| 1.0 | 2025-02-03 | Initial version | Nathan Green, Worldwide Data Architecture Team, Teradata |

---

**End of Domain Module Design Standard**

*This standard defines STRUCTURE and PATTERNS for AI-Native Domain modules. For implementation guidance on temporal tracking, column optimization, and data management approaches, see **"Advocated Data Management Standards"** companion document.*
