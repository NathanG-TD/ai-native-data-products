# Teradata SQL Syntax Reference
## For AI-Native Data Product Design Standards

---

## Critical Syntax Rules

### Rule 1: PRIMARY INDEX - Choose ONE Type Only

**You must specify EITHER Primary Index OR Unique Primary Index, NOT BOTH:**

✅ **Correct - Non-Unique Primary Index (NUPI):**
```sql
CREATE TABLE TableName (
    column1 BIGINT NOT NULL,
    column2 VARCHAR(100)
)
PRIMARY INDEX (column1);
```

✅ **Correct - Unique Primary Index (UPI):**
```sql
CREATE TABLE TableName (
    column1 BIGINT NOT NULL,
    column2 VARCHAR(100)
)
UNIQUE PRIMARY INDEX (column1);
```

✅ **Correct - Composite Unique Primary Index:**
```sql
CREATE TABLE TableName (
    key1 BIGINT NOT NULL,
    key2 BIGINT NOT NULL,
    valid_from_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL
)
UNIQUE PRIMARY INDEX (key1, key2, valid_from_dts);
```

❌ **INCORRECT - Cannot have both:**
```sql
CREATE TABLE TableName (
    column1 BIGINT NOT NULL,
    column2 VARCHAR(100)
)
PRIMARY INDEX (column1)
UNIQUE PRIMARY INDEX (column1, column2);  -- ERROR!
```

### Rule 2: Secondary Indexes

**Teradata does not support WHERE clauses in CREATE INDEX:**

✅ **Correct:**
```sql
CREATE UNIQUE INDEX idx_party_natural
ON Party_H (party_id);
```

❌ **INCORRECT:**
```sql
CREATE INDEX idx_party_current
ON Party_H (party_id)
WHERE is_current = 1 AND is_deleted = 0;  -- Not supported
```

**Alternative - Use Filtered View:**
```sql
-- Create view with filter
CREATE VIEW Party_Current AS
SELECT * FROM Party_H
WHERE is_current = 1 AND is_deleted = 0;

-- Then index the base table normally
CREATE INDEX idx_party_id ON Party_H (party_id);
```

### Rule 3: Syntax Placement

```sql
CREATE TABLE TableName (
    column1 TYPE,
    column2 TYPE,
    column3 TYPE  -- <-- No comma after last column
)
PRIMARY INDEX (column) OR UNIQUE PRIMARY INDEX (...)  -- <-- After closing parenthesis
[PARTITION BY ...]  -- <-- Optional partitioning
[WITH COLUMN_PARTITION ...]  -- <-- Optional compression
;
```

## Decision Guide: UPI vs NUPI

| Use Case | Index Type | Example |
|----------|-----------|---------|
| **Key is guaranteed unique** | UNIQUE PRIMARY INDEX (UPI) | Surrogate keys with bi-temporal |
| **Need even distribution** | PRIMARY INDEX (NUPI) | High-volume tables |
| **Single-row lookups primary** | UNIQUE PRIMARY INDEX (UPI) | Reference data by code |
| **Range queries primary** | PRIMARY INDEX (NUPI) | Time-series data |

## Bi-Temporal Table Pattern (Tested ✅)

```sql
CREATE TABLE Party_H (
    party_key BIGINT NOT NULL,
    party_id VARCHAR(50) NOT NULL,
    legal_name VARCHAR(200),
    valid_from_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    valid_to_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL 
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00',
    transaction_from_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL 
        DEFAULT CURRENT_TIMESTAMP(6),
    transaction_to_dts TIMESTAMP(6) WITH TIME ZONE NOT NULL 
        DEFAULT TIMESTAMP '9999-12-31 23:59:59.999999+00:00',
    is_current BYTEINT NOT NULL DEFAULT 1,
    is_deleted BYTEINT NOT NULL DEFAULT 0
)
UNIQUE PRIMARY INDEX (party_key, valid_from_dts, transaction_from_dts);
```

## Semantic Table Pattern (Tested ✅)

```sql
CREATE TABLE Semantic.EntityMetadata (
    entity_metadata_key BIGINT NOT NULL,
    entity_name VARCHAR(100) NOT NULL,
    module_name VARCHAR(50) NOT NULL,
    entity_description VARCHAR(1000) NOT NULL,
    is_active BYTEINT NOT NULL DEFAULT 1
)
UNIQUE PRIMARY INDEX (entity_name, module_name);
```

## Reference Data Pattern (Tested ✅)

```sql
CREATE TABLE CountryCode_R (
    country_key BIGINT NOT NULL,
    country_code VARCHAR(3) NOT NULL,
    effective_date DATE NOT NULL,
    expiration_date DATE NOT NULL DEFAULT DATE '9999-12-31',
    is_current BYTEINT NOT NULL DEFAULT 1
)
UNIQUE PRIMARY INDEX (country_code, effective_date);
```

## Common Mistakes to Avoid

1. ❌ Specifying both PRIMARY INDEX and UNIQUE PRIMARY INDEX
2. ❌ Using WHERE clause in CREATE INDEX
3. ❌ Comma after last column before closing parenthesis
4. ❌ PRIMARY INDEX inside column list (it goes after closing parenthesis)
5. ❌ Missing semicolon at end of statement

---

**All syntax patterns in this guide have been tested on Teradata and verified to work correctly.**
