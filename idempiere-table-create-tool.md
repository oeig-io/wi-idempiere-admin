---
name: idempiere-table-create
description: Create new tables in iDempiere using the REST API including AD_Table record creation, table completion, and window generation
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-table-create-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Table Create Tool

The purpose of this document is to describe how to create new tables in iDempiere using the REST API.

## Prerequisites

- REST API access via System API Access role
- See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) for authentication pattern

## What You Need From Me

Before executing this tool, provide:

| Field | Description | Example |
|-------|-------------|---------|
| TableName | Technical DB table name with `{Callsign}_` prefix | `ANS_SlabInventory` |
| Name | Human-readable display name (becomes Window/Tab name) | `Slab Inventory` |
| Description | What this table stores | `Tracks slab inventory by warehouse` |
| WindowType | Window type - `M` (Maintain) or `T` (Transaction) | `M` |
| IsSOTrx | Sales transaction window? | `N` |
| IsCreateMenu | Add to application menu? | `Y` |
| AccessLevel | 3=Client+Org, 4=System, 7=All | `3` |

**Special columns** (beyond standard Value, Name, Description, Help):
- DocumentNo, DocAction, DocStatus, Processed (for Transaction type)
- Other custom columns you want to add after initial creation

**Callsign prefix**: Verify with your `deploy.properties` (typically `ANS`).

## Quick Reference

| Step | Action | API/Method |
|------|--------|------------|
| 1 | Create AD_Table record | POST /models/ad_table |
| 2 | Run Create/Complete Table | POST /processes/createtable |
| 3 | Run Synchronize Database | See [idempiere-column-create-tool.md](idempiere-column-create-tool.md) |
| 4 | Run Create Window Tab Field (base tables) | POST /processes/ad_table_createwindow |
| 4 | Create Subtab (link tables) | See [idempiere-subtab-create-tool.md](idempiere-subtab-create-tool.md) |

## Procedure

### Step 1: Create Table Record

```bash
RESPONSE=$(curl -s -X POST "${API_URL}/models/ad_table" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{
        "TableName": "ANS_MyTable",
        "Name": "My Table",
        "Description": "Description of the table",
        "AccessLevel": "3",
        "EntityType": "U",
        "IsView": false,
        "IsDeleteable": true,
        "IsHighVolume": false,
        "IsChangeLog": true,
        "ReplicationType": "L"
    }')

TABLE_ID=$(echo "$RESPONSE" | jq -r '.id')
```

| Field | Description | Example |
|-------|-------------|---------|
| TableName | Technical DB table name | `ANS_SlabInventory` |
| Name | Human-readable display name | `Slab Inventory` |
| AccessLevel | 3=Client+Org, 4=System, 7=All | `3` |
| EntityType | U=User maintained | `U` |

> **📝 Note** - The Name field becomes the Window and Tab display name. Use human-readable text.

### Step 2: Run Create/Complete Table

This process auto-generates standard columns (key, client, org, audit fields, etc.).

```bash
curl -s -X POST "${API_URL}/processes/createtable" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_table\",
        \"record-id\": ${TABLE_ID},
        \"TableName\": \"ANS_MyTable\",
        \"Name\": \"My Table\",
        \"AccessLevel\": \"3\",
        \"EntityType\": \"U\",
        \"IsCreateKeyColumn\": \"Y\",
        \"IsCreateColValue\": \"Y\",
        \"IsCreateColName\": \"Y\",
        \"IsCreateColDescription\": \"Y\",
        \"IsCreateColHelp\": \"Y\",
        \"IsCreateWorkflow\": \"N\",
        \"IsCreateTranslationTable\": \"N\"
    }"
```

**Standard Column Parameters:**

| Parameter | Creates Column | Default |
|-----------|---------------|---------|
| IsCreateKeyColumn | {TableName}_ID (primary key) | Y |
| IsCreateColValue | Value (search key) | Y |
| IsCreateColName | Name | Y |
| IsCreateColDescription | Description | Y |
| IsCreateColHelp | Help | Y |

**Document Table Parameters** (for transactional tables):

| Parameter | Creates Column |
|-----------|---------------|
| IsCreateColDocumentNo | DocumentNo |
| IsCreateColDocAction | DocAction |
| IsCreateColDocStatus | DocStatus |
| IsCreateColProcessed | Processed |

This creates AD_Column records but NOT the database table yet.

### Step 3: Synchronize Database

Get any column ID, then sync. See [idempiere-column-create-tool.md](idempiere-column-create-tool.md) for details.

```bash
COLUMN_ID=$(psqli -t -A -c "SELECT ad_column_id FROM ad_column WHERE ad_table_id = ${TABLE_ID} LIMIT 1;")

curl -s -X POST "${API_URL}/processes/ad_column-sync" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_column\",
        \"record-id\": ${COLUMN_ID}
    }"
```

### Step 4: Create Window, Tab and Field

```bash
curl -s -X POST "${API_URL}/processes/ad_table_createwindow" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_table\",
        \"record-id\": ${TABLE_ID},
        \"IsNewWindow\": \"Y\",
        \"WindowType\": \"M\",
        \"IsSOTrx\": \"N\",
        \"IsCreateMenu\": \"Y\"
    }"
```

| Parameter | Description | Default |
|-----------|-------------|---------|
| IsNewWindow | Create new window | Y |
| WindowType | Window type | M (Maintain) |
| IsSOTrx | Sales transaction | N |
| IsCreateMenu | Add to menu | Y |

## Post-Creation

1. **Grant Role Access** - Add window access to appropriate roles
2. **Reset Cache** - System Admin => Cache Reset
3. **Set Tab Display Mode to Grid** - After creating the window, verify that all tabs are set to Grid view, not Detail. By default, new windows may open in Detail mode. To ensure consistency:
   - Open the Window (AD_Window) record in Application Dictionary
   - For each Tab, find the **Single Row** field and set it to `No` (Grid view)
   - SQL: `UPDATE ad_tab SET issinglerow = 'N' WHERE ad_window_id = {WINDOW_ID};`
   - This is particularly important for material windows that may default to Detail view

## Adding Columns

After initial creation, add columns using [idempiere-column-create-tool.md](idempiere-column-create-tool.md).

For reference types, see the Reference Types table in that document.

## Naming Conventions

Custom tables must use the client callsign prefix from `deploy.properties`.

| Field | Convention | Example |
|-------|------------|---------|
| TableName | `{Callsign}_` prefix, PascalCase | `ANS_SlabInventory` |
| Name | Human-readable, spaces allowed | `Slab Inventory` |

The callsign (e.g., `ANS`) is the tenant's Value/SearchKey in deploy.properties.

## Link Tables

The purpose of this section is to describe how to create link tables in iDempiere. This is important because link tables connect two base tables in a many-to-many relationship and require different configuration than standalone tables.

### What is a Link Table

A link table represents a many-to-many relationship between two base tables. For example, a product can have multiple finishes, and a finish can apply to multiple products.

### Link Table Characteristics

| Characteristic | Configuration |
|----------------|---------------|
| Primary key | Yes - enables change log, attachments |
| Name field | No - use Description instead |
| Standalone window | No - added as subtab to parent windows |
| Standard columns | Skip `Name`, use `Description` |

### Creating a Link Table

Follow the same procedure as base tables with these modifications:

1. **Create Table Record**: Set `IsCreateColName: "N"` to skip the Name column
2. **Add Foreign Key Columns**: Add reference columns to both parent tables after initial creation
3. **Create as Subtab**: Use [idempiere-subtab-create-tool.md](idempiere-subtab-create-tool.md) to add as subtab instead of creating a window

### Example: Material Type Finish Link

For a table linking `ANS_Mat_Type` and `ANS_Mat_Finish`:

```bash
# Create table without Name column
curl -s -X POST "${API_URL}/models/ad_table" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{
        "TableName": "ANS_Mat_Type_Finish",
        "Description": "Links material types with valid finishes",
        "AccessLevel": "3",
        "EntityType": "U"
    }'
```

Then add foreign key columns and create as subtab to Material Types window.

## Process Reference

| Process | Value | Slug |
|---------|-------|------|
| Create/Complete Table | CreateTable | createtable |
| Create Window Tab Field | AD_Table_CreateWindow | ad_table_createwindow |
| Create Fields | AD_Tab_CreateFields | (called internally) |

For Synchronize Column, see [idempiere-column-create-tool.md](idempiere-column-create-tool.md).

Tags: #tool #idempiere #application-dictionary #table-create
