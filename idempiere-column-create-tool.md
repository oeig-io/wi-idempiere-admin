---
name: idempiere-column-create
description: Add columns to existing iDempiere tables using the REST API including AD_Column creation and database synchronization
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-column-create-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Column Create Tool

The purpose of this document is to describe how to add columns to existing tables in iDempiere using the REST API.

## Prerequisites

- REST API access via System API Access role
- See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) for authentication pattern

## Quick Reference

| Step | Action | API/Method |
|------|--------|------------|
| 1 | Create AD_Element (if needed) | POST /models/ad_element |
| 2 | Create AD_Column | POST /models/ad_column |
| 3 | Run Synchronize Database | POST /processes/ad_column-sync |
| 4 | Create AD_Field | POST /models/ad_field |

## Reference Types

| ID | Type | Use For | Length |
|----|------|---------|--------|
| 10 | String | Text | Varies |
| 11 | Integer | Whole numbers | 10 |
| 13 | ID | Primary key | 22 |
| 19 | Table Direct | FK where column = `TableName_ID` exactly | 10 |
| 20 | Yes-No | Boolean | 1 |
| 22 | Number | Decimals | 22 |
| 29 | Quantity | Quantities | 22 |
| 30 | Search | FK with popup (any column name) | 10 |

### Boolean Columns (Yes-No)

> **⚠️ Warning** - Yes-No (boolean) columns must always be `IsMandatory: true` with `DefaultValue: 'N'`. Null boolean states in the database cause system instability and UI issues.

**Naming Convention:** Boolean columns follow iDempiere's standard `IsXxx` pattern without the tenant prefix:

| Pattern | Example | Use For |
|---------|---------|---------|
| `IsXxx` | `IsANSTemplate` | Custom boolean flags (no tenant prefix) |
| `IsXxx` | `IsActive`, `IsSummary` | Standard iDempiere booleans |

**Field Layout:** Checkbox fields require proper layout settings for the label to appear correctly:

```sql
xposition = 2,    -- Horizontal position
columnspan = 2    -- Label appears to right of checkbox
```

### Table Direct vs Search

| Reference | When to Use | AD_Reference_Value_ID |
|-----------|-------------|----------------------|
| Table Direct (19) | Column name = `TableName_ID` exactly (e.g., `M_Product_ID`) | Not needed |
| Search (30) | Column name differs from table (e.g., `AD_Org_Location_ID` → AD_Org) | Required |

**Finding Reference Keys:**
```sql
-- Find validation references for a table
SELECT ad_reference_id, name FROM ad_reference
WHERE name ILIKE '%org%' AND validationtype = 'T';
```

Common reference keys:
| ID | Name | Use For |
|----|------|---------|
| 276 | AD_Org (all) | Any organization |
| 130 | AD_Org (Trx) | Transaction orgs only |
| 138 | C_BPartner (Trx) | Business partners |

## Procedure

### Step 1: Create Element (if needed)

**Create new element when:**
- Column name doesn't exist in AD_Element
- Custom columns must use client callsign prefix (e.g., `ACME_` from deploy.properties)

**Use existing element when:**
- Adding standard iDempiere column (e.g., another `M_Product_ID` reference)

```bash
ELEMENT_ID=$(psqli -t -A -c "SELECT ad_element_id FROM ad_element WHERE columnname = 'AD_Org_Location_ID';")

if [ -z "$ELEMENT_ID" ]; then
    RESPONSE=$(curl -s -X POST "${API_URL}/models/ad_element" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $SESSION_TOKEN" \
        -d '{
            "ColumnName": "AD_Org_Location_ID",
            "Name": "Location Organization",
            "PrintName": "Location Organization",
            "Description": "The organization where this warehouse is physically located",
            "EntityType": "U"
        }')
    ELEMENT_ID=$(echo "$RESPONSE" | jq -r '.id')
fi
```

### Step 2: Create Column

```bash
TABLE_ID=$(psqli -t -A -c "SELECT ad_table_id FROM ad_table WHERE tablename = 'M_Warehouse';")

# Use Search (30) + Reference Key when column name doesn't match TableName_ID
RESPONSE=$(curl -s -X POST "${API_URL}/models/ad_column" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"AD_Table_ID\": ${TABLE_ID},
        \"AD_Element_ID\": ${ELEMENT_ID},
        \"ColumnName\": \"AD_Org_Location_ID\",
        \"Name\": \"Location Organization\",
        \"Description\": \"The organization where this warehouse is physically located\",
        \"AD_Reference_ID\": 30,
        \"AD_Reference_Value_ID\": 276,
        \"FieldLength\": 10,
        \"IsMandatory\": false,
        \"IsUpdateable\": true,
        \"EntityType\": \"U\",
        \"Version\": 1
    }")

COLUMN_ID=$(echo "$RESPONSE" | jq -r '.id')
```

> **📝 Note** - `AD_Org_Location_ID` doesn't match `AD_Org_ID`, so we use Search (30) with Reference Key 276 (AD_Org all).

### Step 3: Synchronize Database

Creates or alters the database column.

```bash
curl -s -X POST "${API_URL}/processes/ad_column-sync" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_column\",
        \"record-id\": ${COLUMN_ID}
    }"
```

> **📝 Note** - Leave DateFrom parameter empty to sync all columns for that table.

### Step 4: Create Field

```bash
TAB_ID=$(psqli -t -A -c "
    SELECT ad_tab_id FROM ad_tab
    WHERE ad_table_id = ${TABLE_ID} AND seqno = 10;")

curl -s -X POST "${API_URL}/models/ad_field" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"AD_Tab_ID\": ${TAB_ID},
        \"AD_Column_ID\": ${COLUMN_ID},
        \"Name\": \"Location Organization\",
        \"Description\": \"The organization where this warehouse is physically located\",
        \"IsDisplayed\": true,
        \"SeqNo\": 100,
        \"IsSameLine\": false,
        \"IsReadOnly\": false,
        \"EntityType\": \"U\"
    }"
```

## SQL Alternative

The purpose of this section is to provide SQL for creating columns in deploy scripts where REST API is not available.

This is important because the REST API cannot properly set reference parameters for FK columns. Using SQL + ad_column-sync ensures iDempiere maintains FK constraints properly.

### Step 1: Create AD_Element

```sql
-- Create AD_Element (if not exists)
INSERT INTO ad_element (
    ad_element_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, columnname, name, printname, description,
    entitytype, ad_element_uu
)
SELECT nextval('ad_element_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'ACME_Mat_Category_ID', 'Category', 'Category',
    'Material category reference',
    'U', uuid_generate_v4()::varchar
WHERE NOT EXISTS (
    SELECT 1 FROM ad_element WHERE columnname = 'ACME_Mat_Category_ID'
)
RETURNING ad_element_id, columnname;
```

### Step 2: Create AD_Column

```sql
-- Create AD_Column (Table Direct for FK)
INSERT INTO ad_column (
    ad_column_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, description, version, entitytype, columnname,
    ad_table_id, ad_reference_id, fieldlength,
    iskey, isparent, ismandatory, isupdateable, isidentifier, seqno,
    istranslated, isencrypted, isselectioncolumn, ad_element_id,
    issyncdatabase, isalwaysupdateable, isautocomplete, isallowlogging,
    isallowcopy, istoolbarbutton, issecure, fkconstrainttype, ishtml,
    ispartitionkey, ad_column_uu
)
SELECT nextval('ad_column_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'Category',
    'Material category reference',
    1, 'U', 'ACME_Mat_Category_ID',
    (SELECT ad_table_id FROM ad_table WHERE tablename = 'ACME_Mat_Type'),
    19,  -- Table Direct
    10,
    'N', 'N', 'N', 'Y', 'N', 0,
    'N', 'N', 'N',
    (SELECT ad_element_id FROM ad_element WHERE columnname = 'ACME_Mat_Category_ID'),
    'N', 'N', 'N', 'Y', 'Y', 'N', 'N', 'N', 'N', 'N',
    uuid_generate_v4()::varchar
WHERE NOT EXISTS (
    SELECT 1 FROM ad_column
    WHERE columnname = 'ACME_Mat_Category_ID'
      AND ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'ACME_Mat_Type')
)
RETURNING ad_column_id, columnname;
```

### Step 3: Sync Database (CRITICAL)

The purpose of running ad_column-sync process is to let iDempiere create the database column and maintain FK constraints properly.

**Do NOT use manual ALTER TABLE** - it bypasses iDempiere's FK constraint management.

```bash
# After AD_Column creation, run sync process
curl -s -X POST "${API_URL}/processes/ad_column-sync" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{
        "model-name": "ad_column",
        "record-id": <COLUMN_ID>
    }'
```

### Step 4: Create AD_Field

> 🔗 **Reference:** See [idempiere-window-tool.md](idempiere-window-tool.md) for field positioning and layout options (XPosition, ColumnSpan, SeqNo).

**Option A: Manual INSERT (Full Control)**

Use SQL INSERT for precise control over individual field properties:

```sql
-- Create AD_Field (if not exists)
INSERT INTO ad_field (
    ad_field_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, description, ad_tab_id, ad_column_id,
    isdisplayed, displaylength, isreadonly, seqno, issameline, isheading,
    isfieldonly, isencrypted, entitytype, ad_field_uu, iscentrallymaintained,
    isdefaultfocus, columnspan
)
SELECT nextval('ad_field_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'Category',
    'Material category reference',
    (SELECT ad_tab_id FROM ad_tab WHERE ad_table_id =
        (SELECT ad_table_id FROM ad_table WHERE tablename = 'ACME_Mat_Type')
        AND seqno = 10),
    (SELECT ad_column_id FROM ad_column
     WHERE columnname = 'ACME_Mat_Category_ID'
       AND ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'ACME_Mat_Type')),
    'Y', 10, 'N', 60, 'N', 'N', 'N', 'N', 'U',
    uuid_generate_v4()::varchar, 'Y', 'N', 2
WHERE NOT EXISTS (
    SELECT 1 FROM ad_field f
    JOIN ad_column c ON f.ad_column_id = c.ad_column_id
    WHERE c.columnname = 'ACME_Mat_Category_ID'
      AND c.ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'ACME_Mat_Type')
);
```

**Option B: Create Fields Process (Bulk)**

For link table subtabs created manually (not via `ad_table_createwindow`), run the Create Fields process to generate all AD_Field records at once:

```bash
curl -s -X POST "${API_URL}/processes/ad_tab_createfields" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_tab\",
        \"record-id\": ${TAB_ID}
    }"
```

**Note:** This process creates fields for standard columns (AD_Client_ID, AD_Org_ID, IsActive, FK columns, PK, UUID) but skips audit columns (Created, CreatedBy, Updated, UpdatedBy) by design. After running, UPDATE field properties as needed to configure parent FK as read-only and hidden in grid view.

## Common Lookups

```sql
-- Find AD_Table_ID
SELECT ad_table_id, tablename FROM ad_table WHERE tablename ILIKE '%warehouse%';

-- Find existing AD_Element
SELECT ad_element_id, columnname, name FROM ad_element WHERE columnname ILIKE '%org%';

-- Find AD_Tab_ID for a window
SELECT t.ad_tab_id, t.name, t.seqno, w.name as window_name
FROM ad_tab t JOIN ad_window w ON t.ad_window_id = w.ad_window_id
WHERE w.name ILIKE '%warehouse%';

-- Find reference keys (validation references)
SELECT ad_reference_id, name FROM ad_reference
WHERE validationtype = 'T' ORDER BY name;
```

## Naming Conventions

Custom columns must use the client callsign prefix from `deploy.properties`.

| Item | Convention | Example |
|------|------------|---------|
| Custom column | `{Callsign}_DescriptiveName_ID` | `ACME_Org_Location_ID` |
| Standard column | Use existing element | `M_Product_ID` |

The callsign (e.g., `ACME`) is the tenant's Value/SearchKey in deploy.properties.

## Post-Creation

1. Run ad_column-sync process to create database column (see Step 3 above)
2. Reset cache: System Admin => Cache Reset
3. Configure field layout: See [idempiere-window-tool.md](idempiere-window-tool.md)

Tags: #tool #idempiere #application-dictionary #column-create
