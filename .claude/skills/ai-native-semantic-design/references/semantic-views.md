# Semantic Views Reference
## Tested on Teradata v20.0 ✅

---

## 1. v_relationship_paths — Multi-Hop Path Discovery (CRITICAL)

This is the primary agent capability in the Semantic module. It enables agents to find JOIN paths between any two tables without knowing the schema in advance. **Do not alter the CTE structure** — it has been validated on Teradata v20.0.

```sql
CREATE VIEW Semantic.v_relationship_paths AS
WITH RECURSIVE relationship_paths (
    source_table,
    target_table,
    path_tables,
    path_joins,
    hop_count,
    path_description
) AS (
    -- Anchor: Forward (1-hop)
    SELECT
        source_table,
        target_table,
        source_table || ' -> ' || target_table          AS path_tables,
        'JOIN ' || target_table || ' ON ' ||
            target_table || '.' || target_column || ' = ' ||
            source_table || '.' || source_column        AS path_joins,
        1                                               AS hop_count,
        relationship_description                        AS path_description
    FROM Semantic.table_relationship
    WHERE is_active = 'Y'

    UNION ALL

    -- Anchor: Reversed (1-hop backward)
    SELECT
        target_table                                    AS source_table,
        source_table                                    AS target_table,
        target_table || ' -> ' || source_table          AS path_tables,
        'JOIN ' || source_table || ' ON ' ||
            source_table || '.' || source_column || ' = ' ||
            target_table || '.' || target_column        AS path_joins,
        1                                               AS hop_count,
        'REVERSE: ' || relationship_description         AS path_description
    FROM Semantic.table_relationship
    WHERE is_active = 'Y'

    UNION ALL

    -- Recursive: Forward extension
    SELECT
        rp.source_table,
        tr.target_table,
        rp.path_tables || ' -> ' || tr.target_table    AS path_tables,
        rp.path_joins || ' | ' ||
        'JOIN ' || tr.target_table || ' ON ' ||
            tr.target_table || '.' || tr.target_column || ' = ' ||
            tr.source_table || '.' || tr.source_column  AS path_joins,
        rp.hop_count + 1                               AS hop_count,
        rp.path_description || ' -> ' || tr.relationship_description AS path_description
    FROM relationship_paths rp
    INNER JOIN Semantic.table_relationship tr
        ON tr.source_table = rp.target_table
       AND tr.is_active = 'Y'
    WHERE rp.hop_count < 5
      AND rp.path_tables NOT LIKE '%' || tr.target_table || '%'

    UNION ALL

    -- Recursive: Backward extension
    SELECT
        rp.source_table,
        tr.source_table                                AS target_table,
        rp.path_tables || ' -> ' || tr.source_table   AS path_tables,
        rp.path_joins || ' | ' ||
        'JOIN ' || tr.source_table || ' ON ' ||
            tr.source_table || '.' || tr.source_column || ' = ' ||
            tr.target_table || '.' || tr.target_column  AS path_joins,
        rp.hop_count + 1                               AS hop_count,
        rp.path_description || ' -> REVERSE: ' || tr.relationship_description AS path_description
    FROM relationship_paths rp
    INNER JOIN Semantic.table_relationship tr
        ON tr.target_table = rp.target_table
       AND tr.is_active = 'Y'
    WHERE rp.hop_count < 5
      AND rp.path_tables NOT LIKE '%' || tr.source_table || '%'
)
SELECT * FROM relationship_paths;

COMMENT ON VIEW Semantic.v_relationship_paths IS
'Multi-hop relationship path discovery - bidirectional traversal up to 5 hops with loop detection.
 Returns complete JOIN syntax for any source→target table pair.
 Agent usage: WHERE source_table = ''A'' AND target_table = ''B'' ORDER BY hop_count.
 Always use QUALIFY ROW_NUMBER() OVER (ORDER BY hop_count) = 1 to get the shortest path.';
```

### Agent Usage Pattern

```sql
-- Find shortest path from Party_H to Transaction_H
SELECT hop_count, path_tables, path_joins
FROM Semantic.v_relationship_paths
WHERE source_table = 'Party_H'
  AND target_table = 'Transaction_H'
ORDER BY hop_count
QUALIFY ROW_NUMBER() OVER (ORDER BY hop_count) = 1;

-- Result (2-hop example):
-- hop_count: 2
-- path_tables: Party_H -> PartyProduct_H -> Transaction_H
-- path_joins:  JOIN PartyProduct_H ON PartyProduct_H.party_key = Party_H.party_key |
--              JOIN Transaction_H ON Transaction_H.party_product_key = PartyProduct_H.party_product_key
```

### How to Use the path_joins Output in Generated SQL

```sql
-- Agent extracts path_joins, splits on ' | ' separator, builds final query:
SELECT p.party_id, p.legal_name, t.transaction_amount, t.transaction_dts
FROM Party_H p
JOIN PartyProduct_H pp  ON pp.party_key = p.party_key
JOIN Transaction_H t    ON t.party_product_key = pp.party_product_key
WHERE p.is_current = 1 AND p.is_deleted = 0
  AND pp.is_current = 1 AND pp.is_deleted = 0
  AND t.is_current = 1 AND t.is_deleted = 0;
-- Note: agent appends is_current/is_deleted filters from entity_metadata
```

---

## 2. v_entity_catalog — Module-Level Entity Discovery

```sql
CREATE VIEW Semantic.v_entity_catalog AS
SELECT
    em.module_name,
    em.entity_name,
    em.entity_description,
    em.table_name,
    em.view_name,
    em.surrogate_key_column,
    em.natural_key_column,
    em.temporal_pattern,
    em.industry_standard,
    dpm.database_name
FROM Semantic.entity_metadata em
LEFT JOIN Semantic.data_product_map dpm
    ON dpm.module_name = em.module_name
   AND dpm.is_active = 'Y'
WHERE em.is_active = 'Y'
ORDER BY em.module_name, em.entity_name;

COMMENT ON VIEW Semantic.v_entity_catalog IS
'Entity catalog joined with module locations - agents use this for complete discovery.
 Returns entity name, physical location, access view, and key column names in one query.';
```

---

## 3. v_entity_schema — Column Structure for a Given Table

```sql
CREATE VIEW Semantic.v_entity_schema AS
SELECT
    cm.database_name,
    cm.table_name,
    cm.column_name,
    cm.business_description,
    cm.data_type,
    cm.is_required,
    cm.is_pii,
    cm.is_sensitive,
    cm.data_classification,
    cm.allowed_values_json
FROM Semantic.column_metadata cm
WHERE cm.is_active = 'Y'
ORDER BY cm.database_name, cm.table_name, cm.column_name;

COMMENT ON VIEW Semantic.v_entity_schema IS
'Column schema view - agents filter by database_name + table_name to understand column structure.
 Includes governance flags (PII, sensitive, classification) for access control decisions.';
```

---

## 4. Agent Discovery Query Sequence

A well-designed agent uses this sequence to understand and query any data product autonomously:

```sql
-- Step 1: Discover all deployed modules
SELECT module_name, database_name, primary_tables, primary_views, deployment_status
FROM Semantic.data_product_map
WHERE is_active = 'Y'
ORDER BY module_name;

-- Step 2: Discover all entities across all modules
SELECT module_name, entity_name, table_name, view_name,
       surrogate_key_column, natural_key_column, temporal_pattern
FROM Semantic.v_entity_catalog;

-- Step 3: Understand naming conventions
SELECT standard_type, standard_value, meaning, examples
FROM Semantic.naming_standard
WHERE is_active = 'Y'
ORDER BY standard_type, standard_value;

-- Step 4: Find how to join two tables (agent constructs query dynamically)
SELECT hop_count, path_tables, path_joins
FROM Semantic.v_relationship_paths
WHERE source_table = '{TableA}'
  AND target_table = '{TableB}'
ORDER BY hop_count
QUALIFY ROW_NUMBER() OVER (ORDER BY hop_count) = 1;

-- Step 5: Understand column semantics for a specific table
SELECT column_name, business_description, is_pii, data_classification
FROM Semantic.v_entity_schema
WHERE database_name = '{DatabaseName}'
  AND table_name = '{TableName}';
```
