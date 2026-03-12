# Teradata COMMENT Syntax Guide

## Verified Syntax Patterns

Based on Teradata documentation and validated examples:

### Table Comments

```sql
COMMENT ON TABLE table_name IS 'comment text';
```

**Example**:
```sql
COMMENT ON TABLE Party_H IS 'Party entity history table with bi-temporal tracking';
```

### Column Comments  

```sql
COMMENT ON COLUMN table_name.column_name IS 'comment text';
```

**Example**:
```sql
COMMENT ON COLUMN Party_H.party_key IS 'Surrogate key - system generated, never reused';
COMMENT ON COLUMN Party_H.party_id IS 'Natural business identifier from source CRM system';
```

### Alternative Syntax (Also Valid)

**Without "IS" keyword**:
```sql
COMMENT ON table_name 'comment text';
COMMENT ON table_name.column_name 'comment text';
```

**Recommendation**: Use the "IS" keyword version for consistency with SQL standards (PostgreSQL, Oracle pattern)

---

## Inline Column Comments (During CREATE TABLE)

**Teradata does NOT support inline COMMENT during CREATE TABLE**:

```sql
-- This does NOT work in Teradata:
CREATE TABLE test (
    col1 INTEGER COMMENT 'This is a comment'  -- ❌ Not supported
);
```

**Must use separate COMMENT statements after CREATE TABLE**:

```sql
CREATE TABLE test (
    col1 INTEGER,
    col2 VARCHAR(100)
);

COMMENT ON TABLE test IS 'Test table description';
COMMENT ON COLUMN test.col1 IS 'First column description';
COMMENT ON COLUMN test.col2 IS 'Second column description';
```

---

## Best Practice Pattern for Design Standards

**In design standard documents, show**:

```sql
-- 1. CREATE TABLE statement
CREATE TABLE Party_H (
    party_key BIGINT NOT NULL,
    party_id VARCHAR(50) NOT NULL,
    legal_name VARCHAR(200)
)
PRIMARY INDEX (party_key);

-- 2. Table comment
COMMENT ON TABLE Party_H IS 
'Party entity history table - customers, employees, vendors with bi-temporal tracking';

-- 3. Column comments
COMMENT ON COLUMN Party_H.party_key IS 
'Surrogate key - system generated unique identifier, never reused, used for all joins';

COMMENT ON COLUMN Party_H.party_id IS 
'Natural business identifier from source CRM system, used for user queries and reports';

COMMENT ON COLUMN Party_H.legal_name IS 
'Legal registered name of party from government records or business registries';
```

---

## Comment Storage

**Comments are stored in Data Dictionary**:
- Table comments: `DBC.TablesV.CommentString`
- Column comments: `DBC.ColumnsV.CommentString`

**Agents can query comments**:
```sql
-- Get table comment
SELECT CommentString 
FROM DBC.TablesV 
WHERE TableName = 'PARTY_H';

-- Get column comments
SELECT ColumnName, CommentString
FROM DBC.ColumnsV
WHERE TableName = 'PARTY_H';
```

---

## Recommendation for Module Standards Updates

**For each table definition in module standards**:

1. Show CREATE TABLE statement
2. Add COMMENT ON TABLE statement
3. Add COMMENT ON COLUMN for all critical columns (at minimum)

**Example template**:
```sql
CREATE TABLE {EntityName}_H (
    {entity}_key BIGINT NOT NULL,
    {entity}_id VARCHAR(50) NOT NULL,
    -- ... other columns ...
    is_current BYTEINT NOT NULL DEFAULT 1,
    is_deleted BYTEINT NOT NULL DEFAULT 0
)
PRIMARY INDEX ({entity}_key);

COMMENT ON TABLE {EntityName}_H IS 
'{Entity} history table with temporal tracking and soft delete support';

COMMENT ON COLUMN {EntityName}_H.{entity}_key IS 
'Surrogate key - system generated, never reused, used for all internal joins';

COMMENT ON COLUMN {EntityName}_H.{entity}_id IS 
'Natural business identifier from source system, used for user queries';

COMMENT ON COLUMN {EntityName}_H.is_current IS 
'Current version flag - 1=current active version, 0=historical version';

COMMENT ON COLUMN {EntityName}_H.is_deleted IS 
'Soft delete flag - 1=logically deleted, 0=active record';
```

---

## Next Steps

1. Update each module design standard to include COMMENT statements
2. Focus on templates and core examples
3. Demonstrate metadata best practice through our own DDL
4. Ensure consistency across all 6 modules

**This aligns our standards documents with our metadata best practice advocacy.**
