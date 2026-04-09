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

## API Requirement

- Read: [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) for authentication patterns and API usage
- Know: that you must use API system access (not tenant)

## Prerequisites

- Always: check for existing AD_Elements before you begin scripting
- AD_Element records must be created before AD_Column records (see Step 1 below)

## Quick Reference

| Step | Action | API/Method |
|------|--------|------------|
| 1 | Create AD_Element (if needed) | POST /models/ad_element |
| 2 | Create AD_Column | POST /models/ad_column |
| 3 | Run Synchronize Database | POST /processes/ad_column-sync |
| 4 | Create AD_Field | POST /models/ad_field |

## Reference Types

| ID | Type | Use For | UI Behavior | Length |
|----|------|---------|-------------|--------|
| 10 | String | Text | Text field | Varies |
| 11 | Integer | Whole numbers | Number field | 10 |
| 13 | ID | Primary key | Hidden/system | 22 |
| 18 | **Table** | Dropdown - requires reference when convention broken | Dropdown | 10 |
| 19 | **Table Direct** | Dropdown - uses naming convention `TableName_ID` | Dropdown | 10 |
| 20 | Yes-No | Boolean | Checkbox | 1 |
| 22 | Number | Decimals | Number field | 22 |
| 29 | Quantity | Quantities | Number field | 22 |
| 30 | **Search** | Popup with filter - for high-volume tables | Popup dialog | 10 |

### Foreign Key Reference Types (Table, Table Direct, Search)

iDempiere provides three reference types for foreign key relationships. Understanding when to use each is critical for proper UI behavior and performance.

#### Decision Matrix

| Type | ID | Use When | Requires Reference | UI Behavior |
|------|-----|----------|-------------------|-------------|
| **Table Direct** | 19 | Column name follows `TableName_ID` convention exactly | **No** - convention auto-resolves | Dropdown |
| **Table** | 18 | Dropdown needed but convention is broken | **Yes** - AD_Reference_Value_ID required | Dropdown |
| **Search** | 30 | High-volume tables (thousands+ records) OR convention broken | Optional - when convention broken | Popup with filter/choose |

#### Table Direct (19) - Convention-Based

Use when the column name matches the target table name exactly with `_ID` suffix.

**Examples:**
```sql
-- M_Product_ID → automatically resolves to M_Product table
AD_Reference_ID = 19
AD_Reference_Value_ID = NULL  -- Not needed!

-- C_BPartner_ID → automatically resolves to C_BPartner table  
AD_Reference_ID = 19
AD_Reference_Value_ID = NULL  -- Not needed!

-- OEIG_Goal_ID → automatically resolves to OEIG_Goal table
AD_Reference_ID = 19
AD_Reference_Value_ID = NULL  -- Not needed!
```

**Key Point:** iDempiere strips the `_ID` suffix and looks for a matching table name. No reference value required!

#### Table (18) - Reference-Based Dropdown

Use when you need a dropdown but the column name breaks convention.

**When to use:**
- Column name doesn't match table name (e.g., `AD_Org_Location_ID` → AD_Org)
- Self-referencing foreign keys (parent-child hierarchies)
- Dynamic dropdowns from custom tables

**Examples:**
```sql
-- AD_Org_Location_ID references AD_Org table (name mismatch)
AD_Reference_ID = 18  -- Table
AD_Reference_Value_ID = 276  -- Required! Reference to AD_Org table

-- OEIG_Goal_Parent_ID self-reference (needs explicit reference)
AD_Reference_ID = 18  -- Table
AD_Reference_Value_ID = 1000006  -- Required! Reference to OEIG_Goal table
```

**Creating Table References:**
When using Table (18), you must first create a Table validation reference:

```sql
-- Step 1: Create AD_Reference (validation type = 'T')
INSERT INTO ad_reference (
    ad_reference_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, validationtype, entitytype, ad_reference_uu
)
SELECT nextval('ad_reference_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'TableName', 'T', 'U', gen_random_uuid()
WHERE NOT EXISTS (SELECT 1 FROM ad_reference WHERE name = 'TableName' AND validationtype = 'T');

-- Step 2: Create AD_Ref_Table (maps to actual table - NO ID COLUMN!)
INSERT INTO ad_ref_table (
    ad_reference_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, ad_table_id, ad_key, ad_display,
    whereclause, orderbyclause, entitytype, ad_ref_table_uu
)
SELECT 
    (SELECT ad_reference_id FROM ad_reference WHERE name = 'TableName' AND validationtype = 'T'),
    0, 0, 'Y', now(), 100, now(), 100,
    (SELECT ad_table_id FROM ad_table WHERE tablename = 'TableName'),
    (SELECT ad_column_id FROM ad_column WHERE ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'TableName') AND iskey = 'Y'),
    (SELECT ad_column_id FROM ad_column WHERE ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'TableName') AND columnname = 'Name'),
    NULL,  -- Optional filter (e.g., 'IsSummary=''Y''')
    'Value',  -- Order by column
    'U',
    gen_random_uuid()
WHERE NOT EXISTS (SELECT 1 FROM ad_ref_table WHERE ad_reference_id = (SELECT ad_reference_id FROM ad_reference WHERE name = 'TableName' AND validationtype = 'T'));
```

**Important:** AD_Ref_Table uses `ad_reference_id` as its primary key - there is NO `ad_ref_table_id` column!

#### Search (30) - Popup with Filtering

Use for high-volume tables where users need to search/filter before selecting.

**When to use:**
- Tables with thousands+ of records (C_BPartner, M_Product on large systems)
- When users need advanced filtering before selection
- Convention-broken FKs that also need search capability

**Examples:**
```sql
-- C_BPartner_ID on high-volume system (use Search instead of Table Direct)
AD_Reference_ID = 30  -- Search
AD_Reference_Value_ID = 138  -- Optional - only if convention broken

-- Product search with filtering capability
AD_Reference_ID = 30
AD_Reference_Value_ID = NULL  -- Not needed if column = M_Product_ID
```

**UI Behavior:**
- **Table/Table Direct (18/19):** Shows dropdown list immediately
- **Search (30):** Shows text field with lookup button → opens popup with filter grid

### Common Reference Keys

```sql
-- Find validation references for a table
SELECT ad_reference_id, name FROM ad_reference
WHERE name ILIKE '%org%' AND validationtype = 'T';
```

| ID | Name | Use For |
|----|------|---------|
| 276 | AD_Org (all) | Any organization |
| 130 | AD_Org (Trx) | Transaction orgs only |
| 138 | C_BPartner (Trx) | Business partners |

### Choosing Between Reference Types

**Quick Decision Guide:**

1. **Does column name match `TableName_ID` exactly?**
   - Yes → Use **Table Direct (19)**
   - No → Continue to question 2

2. **Is this a high-volume table (1000+ records)?**
   - Yes → Use **Search (30)**
   - No → Use **Table (18)**

3. **Do you need search/filter capability?**
   - Yes → Use **Search (30)**
   - No → Use **Table (18)** or **Table Direct (19)**

**Examples:**

```sql
-- Convention match + low volume = Table Direct
OEIG_Goal_ID → Table Direct (19), no reference needed

-- Convention broken + low volume = Table
AD_Org_Location_ID → Table (18), reference to AD_Org required

-- Convention match + high volume = Search (for better UX)
C_BPartner_ID on Order header → Search (30), no reference needed

-- Convention broken + high volume = Search with reference
Custom_External_BPartner_ID → Search (30), reference to C_BPartner required
```

## Table Validation References

**When to Use:**
- Search/Table references to custom tables (not standard iDempiere tables)
- Self-referencing foreign keys (parent-child hierarchies)
- Dynamic dropdowns populated from custom tables

**Architecture:**
Two tables work together:
1. **AD_Reference** - Defines validation type = 'T' (Table)
2. **AD_Ref_Table** - Links to actual table and columns

**Critical Column Structure:**
Unlike most iDempiere tables, `ad_ref_table` uses `ad_reference_id` as its primary key - there is NO separate `ad_ref_table_id` column!

**Step-by-Step Pattern:**

1. **Create AD_Reference:**
```sql
INSERT INTO ad_reference (
    ad_reference_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, validationtype, entitytype, ad_reference_uu
)
SELECT nextval('ad_reference_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'TableName', 'T', 'U', gen_random_uuid()
WHERE NOT EXISTS (SELECT 1 FROM ad_reference WHERE name = 'TableName' AND validationtype = 'T');
```

2. **Create AD_Ref_Table (CRITICAL: No ID column!):**
```sql
INSERT INTO ad_ref_table (
    ad_reference_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, ad_table_id, ad_key, ad_display,
    whereclause, orderbyclause, entitytype, ad_ref_table_uu
)
SELECT 
    (SELECT ad_reference_id FROM ad_reference WHERE name = 'TableName' AND validationtype = 'T'),
    0, 0, 'Y', now(), 100, now(), 100,
    (SELECT ad_table_id FROM ad_table WHERE tablename = 'TableName'),
    (SELECT ad_column_id FROM ad_column WHERE ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'TableName') AND iskey = 'Y'),
    (SELECT ad_column_id FROM ad_column WHERE ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'TableName') AND columnname = 'Name'),
    NULL,  -- Optional filter (e.g., 'IsSummary=''Y''')
    'Value',  -- Order by column
    'U',
    gen_random_uuid()
WHERE NOT EXISTS (SELECT 1 FROM ad_ref_table WHERE ad_reference_id = (SELECT ad_reference_id FROM ad_reference WHERE name = 'TableName' AND validationtype = 'T'));
```

3. **Use in AD_Column:**
```sql
-- For Search reference type
ad_reference_id = 30
ad_reference_value_id = (SELECT ad_reference_id FROM ad_reference WHERE name = 'TableName')
```

**Common Mistakes:**
- **DO NOT** include `ad_ref_table_id` column in INSERT - it doesn't exist!
- **DO NOT** use `nextval('ad_ref_table_sq')` - not needed!
- Ensure column lookups (ad_key, ad_display) reference columns that actually exist
- AD_Ref_Table record must exist before column can use the reference

**Verification:**
```sql
-- Check reference and table validation setup
SELECT r.ad_reference_id, r.name, rt.ad_table_id, rt.ad_key, rt.ad_display, rt.whereclause
FROM ad_reference r
LEFT JOIN ad_ref_table rt ON r.ad_reference_id = rt.ad_reference_id
WHERE r.validationtype = 'T' AND r.name = 'TableName';
```

## Dynamic Validation (AD_Val_Rule)

Create SQL-based validation rules that filter dropdown options based on context.

### Creating a Validation Rule

```sql
INSERT INTO ad_val_rule (
    ad_val_rule_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, description, type, code, entitytype, ad_val_rule_uu
)
SELECT nextval('ad_val_rule_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'Rule Name',
    'Description',
    'S',  -- SQL type
    'ColumnName=@ColumnName@',
    'U',
    gen_random_uuid()
WHERE NOT EXISTS (SELECT 1 FROM ad_val_rule WHERE name = 'Rule Name');
```

### Using Context Variables

Reference parent window fields with `@ColumnName@`:
```sql
M_Product_ID=@M_Product_ID@
C_Project.C_Project_ID=@C_Project_ID@
```

### Handling NULL Context Variables

**CRITICAL:** When the context variable can be NULL/empty, use COALESCE:
```sql
COALESCE(@ANS_AttributeSetInstance_ID@,0)=0 OR M_Warehouse.M_Warehouse_ID IN (...)
```

This prevents "invalid input syntax for type integer" errors when comparing numeric IDs to empty strings.

### Finding Examples

Query existing validation rules:
```sql
SELECT name, code 
FROM ad_val_rule 
WHERE type = 'S' 
  AND code LIKE '%@%';
```

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
- Element already exists in the system

| Scenario | Action |
|----------|--------|
| Element exists (standard iDempiere) | Use existing - do not create (e.g., AD_Table_ID, Record_ID, M_Product_ID) |
| Element exists (custom) | Use existing - do not create duplicate |
| Element does not exist | Create with client callsign prefix |

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

NOTE: This section is a last resort! This section should only be used when the API is not available. The reason the API is preferred is that idempiere handles all the underlying database column creation details (column type, constraints, foreign keys, etc...). Doing so ensures iDempiere's AD_Column records are always in sync with the underlying columns.

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
    'U', uuid_generate_v4()
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
    uuid_generate_v4()
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
    isdisplayed, displaylength, isreadonly, seqno, isheading,
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
    'Y', 10, 'N', 60, 'N', 'N', 'N', 'U',
    uuid_generate_v4(), 'Y', 'N', 2
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
- Field layout: xposition = 2, columnspan = 1 for proper checkbox display
- Always set SeqNoGrid to match or follow SeqNo for grid view column order

## Post-Creation

1. Run ad_column-sync process to create database column (see Step 3 above)
2. Reset cache: System Admin => Cache Reset
3. Configure field layout: See [idempiere-window-tool.md](idempiere-window-tool.md)

## Column SQL (Virtual Columns)

The purpose of this section is to describe Column SQL, which creates virtual computed columns in iDempiere.

This is important because virtual columns allow you to display calculated values without storing them in the database, reducing storage and ensuring the value is always current.

### Overview

Column SQL is stored in the `ColumnSQL` field of `AD_Column`. It defines a SQL expression that iDempiere evaluates when querying data. Unlike physical columns, virtual columns do not require database synchronization since no database column is created.

> **📝 Note** - Virtual columns are always read-only in the UI. Users cannot edit computed values.

### Column SQL Types

iDempiere supports three types of virtual columns based on the prefix used:

| Type | Prefix | SQL Included | Use Case |
|------|--------|--------------|----------|
| Virtual DB | None | Yes - in SELECT query | Computed from database |
| Virtual UI | `@SQL=` | No - replaced with NULL | Lazy loaded in UI only |
| Virtual Search | `@SQLFIND=` | No - replaced with NULL | Search/lookup scenarios |

### Virtual DB Column (Plain SQL)

Use when you want the computed value included in database queries.

```sql
-- Example: Concatenated display value
'(COALESCE(Name, '') || '' - '' || COALESCE(Value, ''))'
```

**Behavior:**
- SQL is included in SELECT queries
- Requires existing database column for the base table
- Can reference other columns in the same table

### Virtual UI Column (@SQL=)

Use when you want the value computed only in the UI, not in database queries. The `@SQL=` prefix indicates lazy loading - the SQL is NOT executed at the database level.

```sql
-- Example: Dynamic status based on conditions
'@SQL= CASE WHEN IsActive=''Y'' THEN ''Active'' ELSE ''Inactive'' END'
```

**Behavior:**
- Database query returns `NULL` for this column
- SQL is parsed and executed only when displaying the field in the UI
- Ideal for values that don't need to be queried or filtered

**Key insight from iDempiere source (POInfo.java):**
```java
if (ColumnSQL.startsWith("@SQL=") || ColumnSQL.startsWith("@SQLFIND="))
    return "NULL AS " + ColumnName;  // Not included in SQL!
```

### Virtual Search Column (@SQLFIND=)

Use for search/lookup virtual columns that need special handling in search dialogs.

```sql
'@SQLFIND= (SELECT name FROM ad_ref_list WHERE ad_reference_id = @AD_Reference_ID@ AND value = ColumnName)'
```

**Behavior:**
- Similar to @SQL= (not included in SQL queries)
- Used for search/filter scenarios in lookup dialogs

### Creating a Virtual Column

Use the SQL patterns from [SQL Alternative](#sql-alternative) section with these modifications:

1. **AD_Element**: Same pattern - create element for the virtual column name
2. **AD_Column**: Add `ColumnSQL` field with your virtual SQL expression (e.g., `'@SQL=...'` or plain SQL)
3. **AD_Field**: Same pattern - create field to display the virtual column
4. **Skip ad_column-sync**: Virtual columns don't require database synchronization

### Important Constraints

1. **Always read-only**: Virtual columns cannot be edited by users
2. **No foreign key**: Cannot reference other tables via FK constraints
3. **No database sync**: Set `IsSyncDatabase = 'Y'` - the sync process skips columns with ColumnSQL
4. **Context variables**: Use `@ContextVariable@` syntax for runtime values
5. **Performance**: Complex SQL in @SQL= columns executes for every row display - keep expressions simple

### Context Variables

> **⚠️ Warning** - Context variables in @SQL= columns are evaluated at display time. For @SQL= (Virtual UI), the SQL is only executed when the field is rendered in the UI, not when the record is saved.

See [Dynamic Validation (AD_Val_Rule)](#dynamic-validation-ad_val_rule) for context variable syntax and handling NULL values.

### Verification

Check if a column is virtual:

```sql
SELECT columnname, columnsql, 
    CASE 
        WHEN columnsql LIKE '@SQL=%' THEN 'Virtual UI'
        WHEN columnsql LIKE '@SQLFIND=%' THEN 'Virtual Search'
        WHEN columnsql IS NOT NULL AND columnsql <> '' THEN 'Virtual DB'
        ELSE 'Physical'
    END as column_type
FROM ad_column
WHERE columnname = 'ACME_Computed_Total';
```

> **📝 Note** - Virtual columns do not require running the ad_column-sync process since no database column is created.

Tags: #tool #idempiere #application-dictionary #column-create
