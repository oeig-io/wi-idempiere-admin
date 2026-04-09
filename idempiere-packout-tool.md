---
name: idempiere-packout
description: PackOut (2Pack) export workflow for creating portable XML packages in iDempiere for configuration migration between environments
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-packout-tool.md
  category: integration
  scope: idempiere
---

# iDempiere PackOut Tool

The purpose of this document is to provide the workflow and reference for creating PackOut (2Pack) export records in iDempiere.

PackOut produces portable XML packages that can be imported into other environments, avoiding manual SQL scripts for dictionary-level artifacts.

This document references the iDempiere REST API. See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) for authentication patterns and API usage.

## TOC

- [Workflow](#workflow)
- [When to Use PackOut](#when-to-use-packout)
- [Client Selection](#client-selection)
- [Creating PackOut Records](#creating-packout-records)
- [Self-Referencing PackOut (Required)](#self-referencing-packout-required)
- [SQL Quote Escaping](#sql-quote-escaping)
- [Running PackOut via REST API](#running-packout-via-rest-api)
- [Deploy Integration](#deploy-integration)
- [Post-Deployment: Role-Window Access](#post-deployment-role-window-access)
- [Common Table IDs](#common-table-ids)

## Workflow

PackOut records are created interactively in the iDempiere UI, not via SQL scripts. The workflow is:

1. **Create the PackOut record** in iDempiere (window: Pack Out - Create)
2. **Add detail lines** for each artifact to export
3. **Add self-referencing detail line** (REQUIRED for reusable PackOuts)
4. **Export via REST API** to produce a zip file
5. **Add the zip to `deploy/`** in idempiere-golive-deploy

The zip is the only migration artifact. No SQL scripts are created for PackOut records.

Because the PackOut includes a self-referencing detail line, the PackOut definition itself is carried to the target environment. This means future modifications are made interactively in either environment, and the target can re-export the same PackOut without recreating it.

## When to Use PackOut

Use PackOut for artifacts stored in AD tables (windows, print formats, print paper, table formats). PackOut handles UUID-based references and idempotent imports automatically.

Avoid the PFT (PrintFormat) detail type. It recursively exports the full table definition (AD_Table, AD_Element, translations) for any table referenced by the print format, producing excessive output. Use Data type instead.

## Client Selection

Choose the correct client based on the artifact's scope:

| Client | Use For | Example |
|--------|---------|---------|
| **System (0)** | Reusable dictionary artifacts | Windows, processes, print formats, custom tables |
| **Tenant** | Tenant-specific data | Payment terms, price lists, chart of accounts |

**System client PackOuts** are imported into all environments and should be used for:
- AD_Window and AD_Tab definitions
- AD_Process definitions
- Print formats and paper settings
- Custom application dictionary elements

**Tenant client PackOuts** are imported only into that specific tenant and should be used for:
- C_PaymentTerm records
- M_PriceList records
- Tenant-specific accounting configurations

## Creating PackOut Records

The SQL below is not used as migration scripts. It serves as a reference for the column names, values, and patterns to use when creating PackOut records interactively in the iDempiere UI.

### AD_Package_Exp

The export package header.

| Column | Value | Notes |
|--------|-------|-------|
| ad_package_type | `X` | Standard export |
| pk_version | `1.0.0` | Semantic version |
| isexportdictionaryentity | `Y` | Required for system-level records |
| isincludeorganizationid | `N` | Typical for system-level |

```sql
INSERT INTO ad_package_exp (
    ad_package_exp_id, ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby,
    name, description, ad_package_type, pk_version,
    processing, processed,
    isexportdictionaryentity, isincludeorganizationid,
    ad_package_exp_uu
) VALUES (
    nextval('ad_package_exp_sq'), 0, 0, 'Y',
    now(), 100, now(), 100,
    'Package Name',
    'Description of what this package contains',
    'X', '1.0.0',
    'N', 'N',
    'Y', 'N',
    uuid_generate_v4()
);
```

### AD_Package_Exp_Detail

Each detail line defines one export artifact.

#### Detail Types

| Type | Name | Use For | Notes |
|------|------|---------|-------|
| D | Data | Table records by SELECT query (preferred for most artifacts) | Use UUIDs in WHERE clauses |
| M | **Menu** | **Menu and Application definitions** | **Preferred for windows** - includes both menu AND window automatically |
| PFT | PrintFormat | Print formats (avoid - exports full table definitions) | Use Data type instead |
| SQL | SQL Statement | Raw SQL to execute on import | Use sparingly |
| W | Window | Window definitions only | Use Menu (M) type instead to get both menu and window |
| P | Process/Report | Process definitions | |
| T | Table | Table definitions | |

> **💡 Best Practice**: When exporting a window that has a menu entry, use type **M (Menu)** instead of type **W (Window)**. The Menu type automatically includes the associated window definition, giving you both the menu item and the window in one detail line.

#### Data Type Detail

The Data type uses a `sqlstatement` containing a full SELECT query. Use UUIDs in WHERE clauses since record IDs change across environments.

```sql
INSERT INTO ad_package_exp_detail (
    ad_package_exp_detail_id, ad_package_exp_id,
    ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby,
    type, line, processing, processed,
    ad_table_id, sqlstatement,
    ad_package_exp_detail_uu
) VALUES (
    nextval('ad_package_exp_detail_sq'), v_exp_id,
    0, 0, 'Y',
    now(), 100, now(), 100,
    'D', 10, 'N', 'N',
    492,  -- ad_table_id for the target table
    'SELECT * FROM AD_PrintPaper WHERE AD_PrintPaper_UU = ''uuid-here''',
    uuid_generate_v4()
);
```

#### Semicolon Syntax for Child Records

Append `; ChildTableName` to the sqlstatement to automatically export child records. The child table must have a column named `ParentTableName_ID` matching the parent.

```sql
-- Parent + child in one line
'SELECT * FROM AD_PrintFormat WHERE AD_PrintFormat_UU = ''uuid-here''; AD_PrintFormatItem'
```

Multiple child chains and deeper hierarchies use additional semicolons and `>`:

```sql
-- Multiple child tables
'SELECT * FROM AD_Package_Exp WHERE AD_Package_Exp_UU = ''uuid-here''; AD_Package_Exp_Detail'

-- Nested hierarchy (grandchildren)
'SELECT * FROM C_Order WHERE C_Order_UU = ''uuid-here''; C_OrderLine > C_OrderLineTax'
```

### Line Ordering

Dependencies must be exported before the records that reference them. Order lines so that supporting records come first.

Example for print format artifacts:

| Line | Table | Why |
|------|-------|-----|
| 10 | AD_PrintPaper | Referenced by print format |
| 20 | AD_PrintTableFormat | Referenced by print format |
| 30 | AD_PrintFormat + items | References paper and table format |
| 40 | AD_Package_Exp + details | Self-referencing (for future re-export) |

## Self-Referencing PackOut (Required)

**⚠️ REQUIRED STEP**: Always include the PackOut record itself as the last detail line. This ensures the PackOut definition is available in target environments for future re-exports.

**Why this matters**: Without a self-referencing line, the PackOut definition exists only in the source environment. If the target environment needs modifications, someone must recreate the entire PackOut from scratch. With self-reference, the PackOut definition travels with the export and can be re-exported from any environment.

**Correct SQL Pattern**:
```sql
-- Get the PackOut UUID first
SELECT ad_package_exp_uu FROM ad_package_exp WHERE ad_package_exp_id = <your_id>;

-- Add as the LAST detail line (highest line number)
INSERT INTO ad_package_exp_detail (...)
VALUES (
    nextval('ad_package_exp_detail_sq'), <packout_id>,
    0, 0, 'Y',
    now(), 100, now(), 100,
    'D', 999, 'N', 'N',  -- High line number (last)
    50005,  -- AD_Package_Exp table
    'SELECT * FROM AD_Package_Exp WHERE AD_Package_Exp_UU = ''your-uuid-here''; AD_Package_Exp_Detail',
    uuid_generate_v4()
);
```

## SQL Quote Escaping

⚠️ **Critical**: The `sqlstatement` column stores SQL with proper single quotes. When inserting via SQL, you must use doubled single quotes (`''`) to escape them.

**Correct Pattern**:
```sql
-- Use '' (two single quotes) to escape ' in the SQL string
-- Example for Menu type (includes window automatically)
'SELECT * FROM AD_Menu WHERE AD_Menu_UU = ''uuid-here'''
```

**Stored Result** (what appears in the column):
```sql
SELECT * FROM AD_Menu WHERE AD_Menu_UU = 'uuid-here'
```

**Common Mistake**: Using double quotes (`"`) instead of doubled single quotes. This causes syntax errors during PackOut execution.

## Running PackOut via REST API

The PackOut process (slug: `packout`, ID: 50004) runs as a system-level call. See "System-Level Calls" and "Granting Process Access" in [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md).

### Using the Generic Script

For convenience, use the generic `run_packout.sh` script in `deploy/lib/`:

```bash
# From inside the iDempiere container
bash /opt/idempiere-deploy/lib/run_packout.sh <packout_id>

# Example
bash /opt/idempiere-deploy/lib/run_packout.sh 1000002
```

The script handles authentication, process access grants, and execution automatically.

### Manual Execution

If you need to call the API directly:

```bash
# After system-level authentication
curl -s -X POST "${API_URL}/processes/packout" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{"model-name":"ad_package_exp","record-id":RECORD_ID}'
```

The response includes the zip file path:

```json
{"summary":"Exported=6 File=/tmp/packout100/Package Name.zip","isError":false}
```

## Deploy Integration

The exported zip goes into the `idempiere-golive-deploy/deploy/` directory with the naming convention:

```
YYYYMMDDHHMMSS_ClientValue_description.zip
```

The `ClientValue` must match `AD_Client.Value` exactly (case-sensitive). Look up the correct value:

```sql
SELECT ad_client_id, value, name FROM ad_client ORDER BY ad_client_id;
```

The system client's `Value` is `SYSTEM` (uppercase), not `System`.

Example: `20260204000000_SYSTEM_slab_label_print_format.zip`

The deploy.sh script batches consecutive zip files and imports them via `RUN_ApplyPackInFromFolder.sh`.

## Post-Deployment: Role-Window Access

After deploying a window via PackOut, you must grant role access so users can see and use the window. PackOut does **not** automatically create role-window access records.

**Table:** AD_Window_Access (ID 201)

**Key Columns:**
- `ad_window_id` - The window being accessed
- `ad_role_id` - The role receiving access
- `ad_client_id` - Must match the role's client
- `isreadwrite` - 'Y' for full access, 'N' for read-only

**Quick Reference SQL:**
```sql
-- Grant window access to multiple roles
INSERT INTO ad_window_access (
    ad_window_id, ad_role_id, ad_client_id, ad_org_id, isactive,
    isreadwrite, created, createdby, updated, updatedby, ad_window_access_uu
) VALUES 
-- Example: Grant Vendor window access to Purchasing roles
(1000011, 1000011, 1000000, 0, 'Y', 'Y', now(), 100, now(), 100, gen_random_uuid()), -- Purchasing Admin
(1000011, 1000010, 1000000, 0, 'Y', 'Y', now(), 100, now(), 100, gen_random_uuid()), -- Purchasing User
(1000011, 1000009, 1000000, 0, 'Y', 'Y', now(), 100, now(), 100, gen_random_uuid()), -- Accounting Admin
(1000011, 1000008, 1000000, 0, 'Y', 'Y', now(), 100, now(), 100, gen_random_uuid())  -- Accounting User
ON CONFLICT (ad_window_id, ad_role_id) DO UPDATE SET
    isactive = 'Y',
    isreadwrite = 'Y',
    updated = now(),
    updatedby = 100;
```

**Notes:**
- The primary key is composite: `(ad_window_id, ad_role_id)`
- Use `ON CONFLICT` for idempotent inserts
- Always match `ad_client_id` to the role's client (not the window's client)
- System windows (client 0) can be accessed by tenant roles

## Common Table IDs

| AD_Table_ID | TableName | Notes |
|-------------|-----------|-------|
| 105 | AD_Window | Use type 'M' (Menu) instead to include both menu and window |
| 106 | AD_Tab | Child records of window |
| 107 | AD_Field | Child records of tab |
| 116 | **AD_Menu** | **Use for windows with menu entries - includes window automatically** |
| **201** | **AD_Window_Access** | **Role-window access grants (see Post-Deployment section)** |
| 489 | AD_PrintFormatItem | |
| 492 | AD_PrintPaper | |
| 493 | AD_PrintFormat | |
| 523 | AD_PrintTableFormat | |
| 50005 | AD_Package_Exp | Self-referencing export |

Tags: #tool #idempiere #packout #2pack #print-format
