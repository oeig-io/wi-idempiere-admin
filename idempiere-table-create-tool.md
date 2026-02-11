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

## Quick Reference

| Step | Action | API/Method |
|------|--------|------------|
| 1 | Create AD_Table record | POST /models/ad_table |
| 2 | Run Create/Complete Table | POST /processes/createtable |
| 3 | Run Synchronize Database | See [idempiere-column-create-tool.md](idempiere-column-create-tool.md) |
| 4 | Run Create Window Tab Field | POST /processes/ad_table_createwindow |

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

## Process Reference

| Process | Value | Slug |
|---------|-------|------|
| Create/Complete Table | CreateTable | createtable |
| Create Window Tab Field | AD_Table_CreateWindow | ad_table_createwindow |
| Create Fields | AD_Tab_CreateFields | (called internally) |

For Synchronize Column, see [idempiere-column-create-tool.md](idempiere-column-create-tool.md).

Tags: #tool #idempiere #application-dictionary #table-create
