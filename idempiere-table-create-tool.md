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

The purpose of this document is to create custom tables in iDempiere with windows for data management.

This is important because ANS business operations require custom material attribute tables (categories, types, finishes, etc.) sourced from Sage that need their own windows.

## Prerequisites

- REST API access via System API Access role
- See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) for authentication pattern

## Quick Reference

| Step | Action | See |
|------|--------|-----|
| 1 | Create AD_Table record | Step 1 below |
| 2 | Run Create/Complete Table | Step 2 below |
| 3 | Add custom columns (optional) | [idempiere-column-create-tool.md](idempiere-column-create-tool.md) |
| 4 | Run Synchronize Database | Step 3 below |
| 5 | Create Window/Tab | Step 4 below |

## Step 1: Create Table Record

```bash
RESPONSE=$(curl -s -X POST "${API_URL}/models/ad_table" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"TableName\": \"${TABLE_NAME}\",
        \"Name\": \"${WINDOW_NAME}\",
        \"Description\": \"Description of the table\",
        \"AccessLevel\": \"3\",
        \"EntityType\": \"U\",
        \"IsDeleteable\": true,
        \"IsChangeLog\": true,
        \"CreatedBy\": {\"id\": ${SYSTEM_USER_ID}},
        \"UpdatedBy\": {\"id\": ${SYSTEM_USER_ID}}
    }")

TABLE_ID=$(echo "$RESPONSE" | grep -o '"id":[0-9]*' | head -1 | cut -d':' -f2)
```

## Step 2: Run Create/Complete Table

Creates standard columns (key, client, org, audit fields) as AD_Column records.

```bash
curl -s -X POST "${API_URL}/processes/createtable" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_table\",
        \"record-id\": ${TABLE_ID},
        \"TableName\": \"${TABLE_NAME}\",
        \"Name\": \"${WINDOW_NAME}\",
        \"AccessLevel\": \"3\",
        \"EntityType\": \"U\",
        \"IsCreateKeyColumn\": \"Y\",
        \"IsCreateColValue\": \"Y\",
        \"IsCreateColName\": \"Y\",
        \"IsCreateColDescription\": \"Y\",
        \"IsCreateColHelp\": \"N\",
        \"IsCreateWorkflow\": \"N\",
        \"IsCreateTranslationTable\": \"N\"
    }"
```

## Step 3: Add Custom Columns (Optional)

For tables with additional columns beyond standard Value/Name/Description, see [idempiere-column-create-tool.md](idempiere-column-create-tool.md).

**Critical**: Add ALL custom columns via SQL first, then sync once at the end (see Step 4).

## Step 4: Synchronize Database

The purpose of this step is to create the physical database table with all columns.

When you call ad_column-sync with any column ID, it creates the physical table containing ALL columns (standard + any custom columns already added via SQL).

```bash
COLUMN_ID=$(psqli -t -A -c "SELECT ad_column_id FROM ad_column WHERE ad_table_id = ${TABLE_ID} AND columnname = 'Value';")

curl -s -X POST "${API_URL}/processes/ad_column-sync" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_column\",
        \"record-id\": ${COLUMN_ID}
    }"
```

> **📝 Note** - Use the 'Value' column ID (or any standard column). This triggers creation of the physical table with ALL columns in one operation.

## Step 5: Create Window, Tab and Field

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

## Post-Creation

1. **Grant Role Access**: Add window access to ANS roles
2. **Set Tab to Grid View**: `UPDATE ad_tab SET issinglerow = 'N' WHERE ad_window_id = {WINDOW_ID};`
3. **Set Org Field Read-Only**: See standard configuration in [idempiere-column-create-tool.md](idempiere-column-create-tool.md)

## Naming Conventions

Custom tables must use the client callsign prefix from `deploy.properties`.

| Field | Convention | Example |
|-------|------------|---------|
| TableName | `{Callsign}_` prefix, PascalCase | `ANS_Mat_Type` |
| Name | Human-readable, spaces allowed | `Material Types` |

## Link Tables

The purpose of this section is to describe how to create link tables. This is important because many-to-many relationships require subtabs, not standalone windows.

See [idempiere-subtab-create-tool.md](idempiere-subtab-create-tool.md) for details.

Tags: #tool #idempiere #application-dictionary #table-create
