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

## Workflow

PackOut records are created interactively in the iDempiere UI, not via SQL scripts. The workflow is:

1. **Create the PackOut record** in iDempiere (window: Pack Out - Create)
2. **Export via REST API** to produce a zip file
3. **Add the zip to `deploy/`** in idempiere-golive-deploy

The zip is the only migration artifact. No SQL scripts are created for PackOut records.

Because the PackOut includes a self-referencing detail line, the PackOut definition itself is carried to the target environment. This means future modifications are made interactively in either environment, and the target can re-export the same PackOut without recreating it.

## When to Use PackOut

Use PackOut for artifacts stored in AD tables (print formats, print paper, table formats). PackOut handles UUID-based references and idempotent imports automatically.

Avoid the PFT (PrintFormat) detail type. It recursively exports the full table definition (AD_Table, AD_Element, translations) for any table referenced by the print format, producing excessive output. Use Data type instead.

## Interactive Reference

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
    uuid_generate_v4()::varchar
);
```

### AD_Package_Exp_Detail

Each detail line defines one export artifact.

#### Detail Types

| Type | Name | Use For |
|------|------|---------|
| D | Data | Table records by SELECT query (preferred for most artifacts) |
| PFT | PrintFormat | Print formats (avoid - exports full table definitions) |
| SQL | SQL Statement | Raw SQL to execute on import |
| W | Window | Window definitions |
| P | Process/Report | Process definitions |
| T | Table | Table definitions |

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
    uuid_generate_v4()::varchar
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

### Self-Referencing PackOut

Include the PackOut record itself as the last detail line. This ensures the PackOut definition is available in target environments for future re-exports.

```sql
-- Line 40: Self-reference with detail lines
'SELECT * FROM AD_Package_Exp WHERE AD_Package_Exp_UU = ''uuid-here''; AD_Package_Exp_Detail'
```

## Client Considerations

All records in a PackOut should belong to the same client. For artifacts that span clients, move them to a single client before exporting. System client (ad_client_id = 0) is preferred for reusable artifacts.

## Running PackOut via REST API

The PackOut process (slug: `packout`, ID: 50004) runs as a system-level call. See "System-Level Calls" and "Granting Process Access" in [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md).

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

## Common Table IDs

| AD_Table_ID | TableName |
|-------------|-----------|
| 489 | AD_PrintFormatItem |
| 492 | AD_PrintPaper |
| 493 | AD_PrintFormat |
| 523 | AD_PrintTableFormat |
| 50005 | AD_Package_Exp |

Tags: #tool #idempiere #packout #2pack #print-format
