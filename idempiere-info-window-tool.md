---
name: idempiere-info-window
description: Info Windows configuration for iDempiere including AD_InfoWindow and AD_InfoColumn setup with filtering and process button integration
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-info-window-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Info Window Tool

The purpose of this document is to provide patterns for creating Info Windows in iDempiere.

For standard INSERT patterns and role access grants, see [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md).

To add process buttons to an Info Window, see [idempiere-process-tool.md](idempiere-process-tool.md).

## AD_InfoWindow

| Column | Value | Notes |
|--------|-------|-------|
| ad_table_id | Table ID | Base table for the window |
| fromclause | Join expression | No SELECT, no WHERE |
| orderbyclause | Column seqno | References AD_InfoColumn.seqno |

```sql
INSERT INTO ad_infowindow (
    ad_infowindow_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, description, ad_table_id, entitytype,
    fromclause, processing, ad_infowindow_uu, isdefault, isdistinct,
    orderbyclause, isvalid, isshowindashboard, maxqueryrecords, isloadpagenum, pagingsize
) VALUES (
    nextval('ad_infowindow_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'Window Name',
    'Description',
    260,  -- ad_table_id
    'U',
    E'M_Product p
LEFT OUTER JOIN M_ProductPrice pr ON (p.M_Product_ID=pr.M_Product_ID AND pr.IsActive=''Y'')
LEFT OUTER JOIN M_AttributeSet pa ON (p.M_AttributeSet_ID=pa.M_AttributeSet_ID)',
    'N', uuid_generate_v4(), 'N', 'N',
    '1',  -- references seqno of sort column
    'Y', 'N', 0, 'Y', 0
);
```

## Fromclause Requirements

### Required Format

iDempiere's `AccessSqlParser` requires specific formatting for fromclause to work correctly.

| Requirement | Description | Example | Required? |
|-------------|-------------|---------|-----------|
| **Parentheses** | ON conditions wrapped in `()` | `ON (t.id=o.fk_id)` | **YES** - Parser fails without these |
| **Multi-line** | One table/join per line | Each JOIN on separate line | Preferred |
| **No spaces** | No spaces around `=` in ON conditions | `ON (t.id=o.fk_id)` NOT `ON (t.id = o.fk_id)` | Recommended |
| **OUTER keyword** | Use `LEFT OUTER JOIN` not `LEFT JOIN` | `LEFT OUTER JOIN` | Preferred but optional |
| **E'' literal** | Use PostgreSQL E'' for newlines | `E'M_Product p\nJOIN...'` | Required for multi-line |

**Critical: Parentheses are mandatory.** The parser looks for `ON (` and expects a closing `)`. Without parentheses, you get the "Could not remove ON" error.

**Note on JOIN keywords:** The parser handles `JOIN`, `INNER JOIN`, `LEFT JOIN`, and `LEFT OUTER JOIN` via regex replacement. While `LEFT OUTER JOIN` is preferred for clarity, `LEFT JOIN` also works if parentheses are present.

### Example: Inventory ASI Info Window

```sql
E'M_AttributeSetInstance t
JOIN M_StorageOnHand soh ON (t.M_AttributeSetInstance_ID=soh.M_AttributeSetInstance_ID AND soh.QtyOnHand>0)
JOIN M_Locator l ON (soh.M_Locator_ID=l.M_Locator_ID)
JOIN M_Warehouse w ON (l.M_Warehouse_ID=w.M_Warehouse_ID)
JOIN M_Product p ON (soh.M_Product_ID=p.M_Product_ID)
LEFT OUTER JOIN M_AttributeSetInstance pasi ON (p.M_AttributeSetInstance_ID=pasi.M_AttributeSetInstance_ID)'
```

**Common Error:** If you see "Could not remove ON" in the log, your fromclause format is incorrect. The parser strips out JOINs it doesn't understand, leaving orphaned table references.

## AD_InfoColumn

### Displayed Column

```sql
INSERT INTO ad_infocolumn (
    ad_infocolumn_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, ad_infowindow_id, ad_element_id, ad_reference_id,
    selectclause, seqno, isqueryafterchange, isidentifier, isdisplayed, isquerycriteria,
    ad_infocolumn_uu, entitytype, columnname, iscentrallymaintained, ismandatory,
    isreadonly, isautocomplete, isrange
) VALUES (
    nextval('ad_infocolumn_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'Column Name',
    v_infowindow_id,
    454,  -- ad_element_id
    30,   -- ad_reference_id
    't.columnname',
    10,   -- seqno (display order)
    'N', 'N',
    'Y',  -- isdisplayed
    'N',  -- isquerycriteria
    uuid_generate_v4(), 'U', 'ColumnName', 'Y', 'N', 'Y', 'N', 'N'
);
```

### Hidden Filter Column

Auto-filters from parent window context. **Both queryoperator and defaultvalue are required.**

```sql
INSERT INTO ad_infocolumn (
    ad_infocolumn_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, ad_infowindow_id, ad_element_id, ad_reference_id,
    selectclause, seqno, isqueryafterchange, isidentifier, isdisplayed, isquerycriteria,
    ad_infocolumn_uu, entitytype, columnname, iscentrallymaintained, ismandatory,
    isreadonly, isautocomplete, isrange,
    queryoperator, defaultvalue
) VALUES (
    nextval('ad_infocolumn_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'Parent Record',
    v_infowindow_id,
    558,  -- ad_element_id for C_Order_ID
    30,
    't.c_order_id',
    70,
    'N', 'N',
    'N',  -- isdisplayed=N (hidden)
    'Y',  -- isquerycriteria=Y (filter)
    uuid_generate_v4(), 'U', 'C_Order_ID', 'Y', 'N', 'Y', 'N', 'N',
    '=',
    '@C_Order_ID@'
);
```

## Gotcha: queryoperator Required

When `isquerycriteria='Y'`, you **must** set `queryoperator`. Missing this causes NPE when opening the Info Window.

Valid values: `=`, `Like`, `>=`, `<=`

## Context Variables in defaultvalue

Use `@ColumnName@` syntax to pull values from the parent window context:

| Pattern | Description |
|---------|-------------|
| `@C_Order_ID@` | Current order ID |
| `@M_Product_ID@` | Current product ID |
| `@#AD_Client_ID@` | Login client ID |
| `@#AD_Org_ID@` | Login org ID |

Tags: #tool #idempiere #info-window
