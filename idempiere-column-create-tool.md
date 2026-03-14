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

The purpose of this document is to describe how to add columns to existing tables in iDempiere.

> **🔗 Reference:** See [idempiere-table-create-tool.md](idempiere-table-create-tool.md) for the complete table creation workflow.

## Prerequisites

- REST API access via System API Access role
- See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) for authentication patterns and API usage
- This document requires System-Level API access. See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) System-Level Calls section for details
- AD_Elements must be created before AD_Columns (see Step 1 below)

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

## Data Type Best Practices

The AD_Reference_ID field in AD_Column determines how data is stored and displayed. Choose reference types based on iDempiere conventions, not just the underlying PostgreSQL type.

### Standard Numeric Types

| Reference ID | Name | AD_Reference_ID Value | Use For |
|-------------|------|---------------------|---------|
| 12 | Amount | 12 | **Default for all monetary and numeric values** - currency amounts, costs, prices, quantities with decimals |
| 37 | Costs+Prices | 37 | Same as Amount (12), use when specifically tracking costs or prices |
| 22 | Number | 22 | General decimals with higher precision - scientific/engineering values |
| 11 | Integer | 11 | Whole numbers only - counters, sequence numbers, IDs |
| 29 | Quantity | 29 | Physical quantities (weight, volume, count with decimals) |

**Default Rule:** Unless specified otherwise, use Reference ID 12 (Amount) for all numeric columns.

**Database Storage:**
All numeric types are stored in PostgreSQL as unbounded `NUMERIC` (no precision or scale limits). The AD_Reference_ID and FieldLength values control iDempiere's display formatting, not database constraints.

**AD_Column FieldLength for Numeric Types:**
- Set FieldLength = 22 for all numeric reference types (12, 22, 29, 37)
- This controls display precision in iDempiere, not database storage
- The database stores full precision without limits

**Why not Integer or plain Number?**
- Integer (11) cannot store decimal values - inappropriate for costs, prices, percentages
- Number (22) stores 10 decimal places which is excessive for business currency and causes display issues
- Amount (12) is the iDempiere standard for all business numeric values

### JSON and JSONB Storage

| Reference ID | Name | AD_Reference_ID Value | PostgreSQL Type | Use For |
|-------------|------|---------------------|-----------------|---------|
| 200267 | JSON | 200267 | JSONB | Configuration data, adapter settings, flexible structured data |

**When to use JSON:**
- Configuration that doesn't need reporting or foreign key constraints
- Adapter settings (like OpenCode CLI parameters)
- Dynamic or evolving data structures
- Data that doesn't require iDempiere validation rules

**When NOT to use JSON:**
- Data that needs foreign key relationships
- Fields that will be used in WHERE clauses for reporting
- Values that need iDempiere's standard validation (mandatory, range checks)
- Amounts, dates, or standard business data

**Important:** AD_Column stores JSON data as text. The ad_column-sync process maps reference 200267 to PostgreSQL's native JSONB type which provides:
- Binary storage (more efficient than JSON text)
- GIN index support for queries
- JSON operators and functions

**Example JSON column:**
```sql
INSERT INTO ad_column (
    ad_column_id, ad_table_id, columnname, name,
    ad_reference_id,  -- Set to 200267 for JSON
    fieldlength,      -- Use 2000 for TEXT storage of JSON
    ...
)
SELECT nextval('ad_column_sq'), v_table_id, 'OEIG_Adapter_Config', 'Adapter Config',
    200267,  -- JSON reference type
    2000,    -- Length for TEXT
    ...
```

### PostgreSQL Type Mapping

AD_Reference_ID determines the PostgreSQL column type created by ad_column-sync:

| AD_Reference_ID | PostgreSQL Type | iDempiere Display | Storage |
|----------------|-----------------|-------------------|---------|
| 10 (String) | VARCHAR(n) | FieldLength determines max length | Variable |
| 11 (Integer) | NUMERIC | No decimals | Unbounded |
| 12 (Amount) | NUMERIC | FieldLength=22, 2 decimal places shown | Unbounded |
| 14 (Text) | TEXT | Unlimited | Variable unlimited |
| 20 (Yes-No) | CHAR(1) | Y/N | Single character |
| 22 (Number) | NUMERIC | FieldLength=22, 10 decimal places shown | Unbounded |
| 29 (Quantity) | NUMERIC | FieldLength=22, 2 decimal places shown | Unbounded |
| 200267 (JSON) | JSONB | Parsed JSON | Binary JSON |

**Important:** PostgreSQL NUMERIC columns have no precision or scale limits. All numeric data is stored with full precision. The AD_Reference_ID and FieldLength only affect how iDempiere displays the values.

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

**Behavior:**
- **Without DateFrom parameter:** Only the specified column ID gets synced
- **With DateFrom parameter:** Syncs specified column plus any columns updated after that date

```bash
# Sync a single column
curl -s -X POST "${API_URL}/processes/ad_column-sync" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_column\",
        \"record-id\": ${COLUMN_ID}
    }"
```

> **📝 Note** - When adding multiple columns to an EXISTING table, you must sync each column individually (one API call per column). Use the pattern below to iterate through all new column IDs.

#### Syncing Multiple New Columns

When adding multiple columns to an existing table, iterate through each new column ID:

```bash
# After creating AD_Columns via SQL, get all new column IDs
COLUMN_IDS=$(psqli -t -A -c "SELECT ad_column_id FROM ad_column WHERE ad_table_id = ${TABLE_ID} AND issyncdatabase = 'N';")

# Sync each column individually
for COL_ID in ${COLUMN_IDS}; do
    echo "Syncing column ID: ${COL_ID}"
    curl -s -X POST "${API_URL}/processes/ad_column-sync" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $SESSION_TOKEN" \
        -d "{
            \"model-name\": \"ad_column\",
            \"record-id\": ${COL_ID}
        }"
done
```

Alternatively, use the DateFrom parameter with a timestamp slightly before your column creation to batch sync all recent changes:

```bash
# Set DateFrom to 5 minutes ago to sync all recently updated columns
DATE_FROM=$(date -d '5 minutes ago' -u +"%Y-%m-%dT%H:%M:%SZ")

curl -s -X POST "${API_URL}/processes/ad_column-sync" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_column\",
        \"record-id\": ${COLUMN_ID},
        \"DateFrom\": \"${DATE_FROM}\"
    }"
```

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
| Custom column | Callsign_DescriptiveName_ID | ACME_Org_Location_ID |
| Standard column | Use existing element | M_Product_ID |
| Boolean column | IsCallsignXxx pattern | IsOEIGTemplate, IsOEIGRequired |

The callsign (e.g., ACME or OEIG) is the tenant's Value/SearchKey in deploy.properties.

Boolean Column Rules:
- Always use IsCallsignXxx pattern (Is + Callsign + Name)
- Must be IsMandatory: true with DefaultValue: 'N'
- Field layout: xposition = 2, columnspan = 2 for proper checkbox display

## Post-Creation

1. Run ad_column-sync process to create database column (see Step 3 above)
2. Reset cache: System Admin => Cache Reset
3. Configure field layout: See [idempiere-window-tool.md](idempiere-window-tool.md)

Tags: #tool #idempiere #application-dictionary #column-create
