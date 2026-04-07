---
name: idempiere-pre-minted-uuid
description: Pre-mint UUIDs at authoring time for stable cross-script entity references in iDempiere deploy scripts and Groovy code
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-pre-minted-uuid-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Pre-Minted UUID

The purpose of this document is to describe the pattern for pre-minting UUIDs at authoring time so that entities created by one script can be reliably referenced by other scripts.

This is important because every iDempiere table has a `_uu` column with a unique index. When scripts use `uuid_generate_v4()` at runtime, each deployment produces different UUIDs — making cross-script references impossible without fragile name-based lookups.

## TOC

- [The Problem](#the-problem)
- [The Pattern](#the-pattern)
- [Registry Format](#registry-format)
- [When to Pre-Mint](#when-to-pre-mint)

## The Problem

Runtime-generated UUIDs force name-based lookups:

```groovy
// Brittle: breaks if someone renames the attribute
def blockAttrId = DB.getSQLValue(trxName,
    "SELECT m_attribute_id FROM m_attribute WHERE name = 'Block' AND ad_client_id IN (0, ?)", clientId)
```

Pre-minted UUIDs enable stable references:

```groovy
// Stable: rename-proof, self-documenting
def blockAttrId = DB.getSQLValue(trxName,
    "SELECT m_attribute_id FROM m_attribute WHERE m_attribute_uu = 'b61d1528-1ae0-47b9-8cd1-666669be5ca7'")
```

## The Pattern

Three steps: **mint**, **assign**, **use**.

### Step 1: Mint

Query the database for valid UUIDs before writing the script:

```sql
SELECT uuid_generate_v4() FROM generate_series(1, 10);
```

### Step 2: Assign

Map each UUID to an entity in the script header as a comment block registry:

```sql
-- =============================================================================
-- Pre-minted UUIDs
-- =============================================================================
-- Attribute Sets
--   Stone Slab:   23c5d798-a057-44f4-834e-7f955b526652
--   Tile:         2638e58f-306f-4957-aa9d-595874f27e1c
-- Attributes
--   Block:        b61d1528-1ae0-47b9-8cd1-666669be5ca7
--   Slab:         bb2edcde-a319-452c-bf19-13fa44580a55
-- =============================================================================
```

### Step 3: Use

**Creating script** — INSERT with the pre-minted UUID:

```sql
INSERT INTO m_attribute (..., m_attribute_uu)
VALUES (..., 'b61d1528-1ae0-47b9-8cd1-666669be5ca7');
```

For idempotent scripts that may run against existing environments, update the UUID in the ELSE branch:

```sql
IF v_attr_id IS NULL THEN
    INSERT INTO m_attribute (..., m_attribute_uu)
    VALUES (..., 'b61d1528-1ae0-47b9-8cd1-666669be5ca7');
ELSE
    UPDATE m_attribute SET m_attribute_uu = 'b61d1528-1ae0-47b9-8cd1-666669be5ca7'
    WHERE m_attribute_id = v_attr_id;
END IF;
```

**Consuming script** — look up by UUID:

```groovy
def blockAttrId = DB.getSQLValue(trxName,
    "SELECT m_attribute_id FROM m_attribute WHERE m_attribute_uu = 'b61d1528-1ae0-47b9-8cd1-666669be5ca7'")
```

## Registry Format

The creating script is the **source of truth**. Place the registry in a comment block at the top of the file. Consuming scripts reference the source:

```groovy
// Pre-minted UUIDs - stable references from 20260215000000_create_all_attribute_sets.sql
```

## When to Pre-Mint

| Scenario | UUID Strategy |
|----------|--------------|
| Entity referenced by another script (Groovy, SQL) | **Pre-mint** |
| Entity referenced only within the same script | `uuid_generate_v4()` is fine |
| Junction/leaf records (m_attributeuse, ad_process_access) | `uuid_generate_v4()` is fine |

Tags: #tool #idempiere #uuid #deploy-pattern
