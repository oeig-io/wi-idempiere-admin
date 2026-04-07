---
name: idempiere-application-dictionary
description: Foundational patterns for configuring iDempiere via Application Dictionary (AD) tables including standard INSERT patterns for AD tables, process relationships, info window structures, and button column configurations
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-application-dictionary-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Application Dictionary Tool

The purpose of this document is to provide foundational patterns for configuring iDempiere via Application Dictionary (AD) tables.

## AD Table Relationships

### Process

```
AD_Rule (Groovy script) → See [idempiere-groovy-deploy-pattern-tool.md](idempiere-groovy-deploy-pattern-tool.md)
    └── AD_Process (process definition)
        └── AD_Process_Para (parameters)
        └── AD_Process_Access (role permissions)
        └── AD_InfoProcess (links to Info Window)
        └── AD_ToolBarButton (links to tab toolbar)
```

See [idempiere-process-tool.md](idempiere-process-tool.md) for process-specific patterns.

### Info Window

```
AD_InfoWindow (window definition)
    └── AD_InfoColumn (columns and filters)
    └── AD_InfoWindow_Access (role permissions)
    └── AD_InfoProcess (links to Process)
```

See [idempiere-info-window-tool.md](idempiere-info-window-tool.md) for info window-specific patterns.

### Button Column

Creates a button field that opens an Info Window when clicked.

```
AD_Element (column metadata)
    └── AD_Column (table column, links to AD_InfoWindow)
        └── AD_Field (window field display)
```

> 🔗 **Reference:** See [idempiere-window-tool.md](idempiere-window-tool.md) for field positioning and layout options.

## Standard INSERT Pattern

All AD table inserts follow this structure:

```sql
INSERT INTO ad_tablename (
    ad_tablename_id,          -- nextval('ad_tablename_sq')
    ad_client_id,             -- 0 (system) or client ID
    ad_org_id,                -- 0 (all orgs) or org ID
    isactive,                 -- 'Y'
    created, createdby,       -- now(), 100
    updated, updatedby,       -- now(), 100
    -- table-specific columns
    ad_tablename_uu           -- uuid_generate_v4()::varchar
) VALUES (...);
```

## Role Access Pattern

Grant access to all active roles (idempotent):

```sql
INSERT INTO ad_OBJECT_access (
    ad_client_id, ad_org_id, ad_OBJECT_id, ad_role_id,
    isactive, created, createdby, updated, updatedby, ad_OBJECT_access_uu
)
SELECT 0, 0,
    (SELECT ad_OBJECT_id FROM ad_OBJECT WHERE name = 'Object Name'),
    ad_role_id, 'Y', now(), 100, now(), 100, uuid_generate_v4()::varchar
FROM ad_role
WHERE isactive = 'Y'
  AND NOT EXISTS (
    SELECT 1 FROM ad_OBJECT_access a
    WHERE a.ad_OBJECT_id = (SELECT ad_OBJECT_id FROM ad_OBJECT WHERE name = 'Object Name')
      AND a.ad_role_id = ad_role.ad_role_id
  );
```

Replace `OBJECT` with: `infowindow`, `process`.

## Button Column Pattern

Creates a button that opens an Info Window. Four steps:

```sql
-- 1. AD_Element
INSERT INTO ad_element (
    ad_element_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, columnname, name, printname, description,
    entitytype, ad_element_uu
) VALUES (
    nextval('ad_element_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'ANS_ButtonName', 'Button Display Name', 'Button Display Name',
    'Description', 'U', uuid_generate_v4()::varchar
);

-- 2. AD_Column (links to Info Window)
INSERT INTO ad_column (
    ad_column_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, description, version, entitytype, columnname,
    ad_table_id, ad_reference_id, fieldlength,
    iskey, isparent, ismandatory, isupdateable, isidentifier, seqno,
    istranslated, isencrypted, isselectioncolumn, ad_element_id,
    issyncdatabase, isalwaysupdateable, isautocomplete, isallowlogging,
    isallowcopy, istoolbarbutton, issecure, fkconstrainttype, ishtml, ispartitionkey,
    ad_infowindow_id, ad_column_uu
) VALUES (
    nextval('ad_column_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'Button Display Name', 'Description', 1, 'U', 'ANS_ButtonName',
    259,  -- ad_table_id
    28,   -- Button reference type
    1, 'N', 'N', 'N', 'Y', 'N', 0, 'N', 'N', 'N',
    (SELECT ad_element_id FROM ad_element WHERE columnname = 'ANS_ButtonName'),
    'N', 'Y',  -- isalwaysupdateable=Y for buttons on completed docs
    'N', 'Y', 'Y', 'N', 'N', 'N', 'N', 'N',
    (SELECT ad_infowindow_id FROM ad_infowindow WHERE name = 'Target Window'),
    uuid_generate_v4()::varchar
);

-- 3. Database column
ALTER TABLE tablename ADD COLUMN ans_buttonname CHAR(1);

-- 4. AD_Field
INSERT INTO ad_field (
    ad_field_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, description, help, ad_tab_id, ad_column_id,
    isdisplayed, displaylogic, displaylength, isreadonly, seqno, sortno,
    isheading, isfieldonly, isencrypted, entitytype, ad_field_uu,
    iscentrallymaintained, isdefaultfocus, numlines, columnspan, xposition
) VALUES (
    nextval('ad_field_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'Button Display Name', 'Description', 'Help text',
    186,  -- ad_tab_id
    (SELECT ad_column_id FROM ad_column WHERE columnname = 'ANS_ButtonName' AND ad_table_id = 259),
    'Y', '@IsSOTrx@=Y & @Processed@=Y',  -- displaylogic optional
    1, 'N', 590, 0, 'N', 'N', 'N', 'U', uuid_generate_v4()::varchar,
    'Y', 'N', 0, 2, 2
);
```

**Gotchas:**
- `isalwaysupdateable='Y'` allows button click on processed/completed documents
- Button columns use `CHAR(1)` unless columnname ends in `_ID` (then `NUMERIC(10)`)
- Toolbar buttons (AD_ToolBarButton) cannot open Info Windows - use this pattern instead

## Reference Types

| ID | Name | Use For |
|----|------|---------|
| 10 | String | Text fields |
| 11 | Integer | Whole numbers |
| 12 | Amount | Currency amounts |
| 14 | Text | Long text |
| 15 | Date | Date only |
| 16 | DateTime | Date and time |
| 17 | List | Dropdown from AD_Ref_List |
| 19 | Table Direct | FK with _ID suffix |
| 20 | Table | FK via AD_Reference validation |
| 28 | Button | Clickable button |
| 29 | Quantity | Quantities |
| 30 | Search | FK with search popup |
| 37 | Costs+Prices | Prices and costs |

## Naming Conventions

| Item | Convention | Example |
|------|------------|---------|
| Custom columns | `ANS_` prefix | `ANS_CreatePOFromSOLines` |
| Sequences | `tablename_sq` | `ad_infowindow_sq` |

## Common Lookups

```sql
-- Find AD_Table_ID
SELECT ad_table_id, tablename FROM ad_table WHERE tablename ILIKE '%order%';

-- Find AD_Element_ID
SELECT ad_element_id, columnname, name FROM ad_element WHERE columnname = 'M_Product_ID';

-- Find AD_Tab_ID
SELECT t.ad_tab_id, t.name, w.name as window_name
FROM ad_tab t JOIN ad_window w ON t.ad_window_id = w.ad_window_id
WHERE w.name ILIKE '%sales order%';

-- Find AD_Reference_ID
SELECT ad_reference_id, name FROM ad_reference WHERE validationtype = 'D' ORDER BY name;
```

## Cache Reset

After AD table changes, reset the application cache:
- Menu: System Admin => Cache Reset
- Or restart the application server

## Reference

Working examples:
- Groovy process: `idempiere-golive-deploy/deploy/20260406214100_mr_line_lot_create_v2.sh`
- SQL-only: `idempiere-golive-deploy/deploy/20260121000000_so_to_po_create_process.sql`

Tags: #tool #idempiere #application-dictionary
