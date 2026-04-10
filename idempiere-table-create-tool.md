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

This is important because ACME business operations require custom material attribute tables (categories, types, finishes, etc.) sourced from external systems that need their own windows.

## Prerequisites

- REST API access via System API Access role
- See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) for authentication patterns and API usage
- This document requires System-Level API access. See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) System-Level Calls section for details
- Review [idempiere-column-create-tool.md](idempiere-column-create-tool.md) - elements must be created before columns

## Quick Reference

| Step | Action | See |
|------|--------|-----|
| 1 | Create AD_Elements for custom columns | [idempiere-column-create-tool.md](idempiere-column-create-tool.md) |
| 2 | Create AD_Table record | Step 2 below |
| 3 | Run Create/Complete Table process | Step 3 below |
| 4 | Add additional custom columns via SQL | [idempiere-column-create-tool.md](idempiere-column-create-tool.md) |
| 5 | Run ad_column-sync process | Step 5 below |
| 6 | Create Window/Tab (optional) | Step 6 below |

## Step 1: Create AD_Elements for Custom Columns

Before creating any table or columns, create AD_Elements for all custom column names that do not already exist. This must be done first because AD_Columns reference AD_Elements.

See the Create Element section in [idempiere-column-create-tool.md](idempiere-column-create-tool.md) for naming conventions including the client callsign prefix.

## Step 2: Create Table Record

Create a minimal AD_Table record. This record should have NO columns yet - they will be added in the next steps.

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

## Step 3: Run Create/Complete Table Process

The `createtable` process creates standard columns on a new AD_Table record. This provides a complete foundation including primary key, UUID, audit fields (Created, CreatedBy, Updated, UpdatedBy), and standard fields (Value, Name, Description, AD_Client_ID, AD_Org_ID, IsActive).

Requirements:
- Target table must have NO columns (fresh AD_Table record only)
- Process automatically creates AD_Elements for the primary key and UUID columns if they don't exist
- Use the boolean flags to control which standard columns are created

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

> **📝 Note** - See Alternative: CopyColumnsFromTable Process section for an alternative approach that copies columns from an existing table instead of creating them from flags.

## Step 4: Add Additional Custom Columns

After the Create/Complete Table process creates the standard columns, add any additional custom columns your table requires. These must reference AD_Elements created in Step 1.

See [idempiere-column-create-tool.md](idempiere-column-create-tool.md) for the column creation procedure.

## Step 5: Synchronize Database

The ad_column-sync process creates the physical database table with proper column types, constraints, and foreign keys. This is the ONLY supported method for creating the physical table - never use manual CREATE TABLE statements.

**For NEW tables:** When you run ad_column-sync with any column ID from the table, iDempiere creates the physical table with ALL AD_Column records in one operation.

**For EXISTING tables:** When adding new columns, you must run ad_column-sync for EACH new column individually, or use the DateFrom parameter to batch sync columns updated after a specific time.

```bash
# For NEW tables - any column works, creates entire table
COLUMN_ID=$(psqli -t -A -c "SELECT ad_column_id FROM ad_column WHERE ad_table_id = ${TABLE_ID} LIMIT 1;")

curl -s -X POST "${API_URL}/processes/ad_column-sync" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_column\",
        \"record-id\": ${COLUMN_ID}
    }"
```

> **📝 Note** - When creating a NEW table, you can use any column ID from the table. The process creates the entire physical table with all columns. When updating EXISTING tables with new columns, see [idempiere-column-create-tool.md](idempiere-column-create-tool.md) for the proper sync pattern.

Generate the window, tab, and field structure automatically:

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

1. Grant Role Access: Add window access to ACME roles
2. Set Tab to Grid View: `UPDATE ad_tab SET issinglerow = 'N' WHERE ad_window_id = {WINDOW_ID};`
3. Set Org Field Read-Only: See standard configuration in [idempiere-column-create-tool.md](idempiere-column-create-tool.md)

## Naming Conventions

Custom tables must use the client callsign prefix from deploy.properties.

| Field | Convention | Example |
|-------|------------|---------|
| TableName | Callsign_ prefix, PascalCase | ACME_Mat_Type |
| Name | Human-readable, spaces allowed | Material Types |

## Link Tables

The purpose of this section is to describe how to create link tables in iDempiere. This is important because many-to-many relationships between base tables require subtabs, not standalone windows, to maintain data integrity and proper navigation.

### Link Table Requirements

| Requirement | Description |
|-------------|-------------|
| Primary key | REQUIRED - created by Create/Complete Table or CopyColumnsFromTable |
| UUID column | REQUIRED - included in standard column creation |
| Name field | OMIT - use Description instead (no standalone window) |
| FK columns | Add via SQL after standard columns are created, before sync |
| Standalone window | NO - added as subtab to parent window |

### Example: Product Price Subtab

The Product window => Price subtab demonstrates the link table pattern:

| Column | Purpose |
|--------|---------|
| M_ProductPrice_ID | Primary key (from IsCreateKeyColumn) |
| M_ProductPrice_UU | UUID for external references |
| M_Product_ID | FK to Product (parent) |
| M_PriceList_Version_ID | FK to Price List Version (parent) |
| PriceList, PriceStd, PriceLimit | Price values |

### Creation Steps

1. Create AD_Elements for any custom FK columns
2. Create table record (no columns yet)
3. Run Create/Complete Table process (or CopyColumnsFromTable for link tables that omit Name/Value)
4. Add FK columns to parent tables via SQL
5. Sync database once at end
6. Add as subtab (not standalone window), then create fields using [idempiere-column-create-tool.md](idempiere-column-create-tool.md) and the Create Fields process

> **📝 Note** - See Product window => Price subtab in Application Dictionary for live example.

## Alternative: CopyColumnsFromTable Process

The purpose of this section is to document the CopyColumnsFromTable process as an alternative to the Create/Complete Table process.

This is important because CopyColumnsFromTable copies columns from an existing source table, which can be useful for link tables or when you need to match the column structure of an existing table.

**Process slug:** `ad_table_copycolumnsfromtable`

**When to consider:**
- Link tables that need a subset of standard columns (e.g., omit Name/Value)
- Tables that should mirror the column structure of an existing table

**Requirements:**
- Target table must have NO columns (fresh AD_Table record only)
- Process automatically creates AD_Elements for the primary key and UUID columns if they don't exist
- All copied columns inherit the target table's EntityType

```bash
# Get source table ID (C_Calendar is a common source for standard columns)
SOURCE_TABLE_ID=$(psqli -t -A -c "SELECT ad_table_id FROM ad_table WHERE tablename = 'C_Calendar';")

curl -s -X POST "${API_URL}/processes/ad_table_copycolumnsfromtable" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d "{
        \"model-name\": \"ad_table\",
        \"record-id\": ${TABLE_ID},
        \"AD_Table_ID\": ${SOURCE_TABLE_ID}
    }"
```

Tags: #tool #idempiere #application-dictionary #table-create
