---
name: idempiere-window
description: Configure iDempiere windows, tabs, and field layouts using AD_Window, AD_Tab, and AD_Field tables with modern positioning controls
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-window-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Window Tool

The purpose of this document is to describe how to configure iDempiere window layouts, tab structures, and field positioning.

This is important because proper window configuration ensures users can efficiently interact with data in both detail (form) and grid views.

## AD Structure

```
AD_Window (window definition)
    └── AD_Tab (tabs within window)
        └── AD_Field (fields within tab)
```

## Field Layout Concepts

### Positioning Fields (Modern Approach)

The modern field layout system uses three key controls:

| Field | Purpose | Values |
|-------|---------|--------|
| SeqNo | Vertical order (top to bottom) | 10, 20, 30... (must be unique per field) |
| XPosition | Horizontal start position | 1 = left column, 2 = right column |
| ColumnSpan | Width in columns | 1 = standard, 2+ = wide |

> ⚠️ **Deprecated:** IsSameLine is legacy and superseded by XPosition/ColumnSpan. Do not use for new layouts.

### Common Layout Patterns

**Two-Column Layout (fields side-by-side):**
```sql
-- Left field
UPDATE ad_field SET seqno = 90, xposition = 1, columnspan = 1 WHERE ad_field_id = 1000102;

-- Right field  
UPDATE ad_field SET seqno = 95, xposition = 2, columnspan = 1 WHERE ad_field_id = 1000096;
```

Result: Two fields share one line in the detail view.

**Full-Width Field:**
```sql
UPDATE ad_field SET seqno = 100, xposition = 1, columnspan = 2 WHERE ad_field_id = 3510;
```

Result: Field spans both columns (useful for Description, notes, etc.).

**Checkbox Field (special case):**
```sql
UPDATE ad_field SET seqno = 60, xposition = 2, columnspan = 2 WHERE ad_field_id = <checkbox_field>;
```

Result: Checkbox appears with label properly aligned to the right.

### Grid vs Detail View

| View | Controlled By | Purpose |
|------|--------------|---------|
| Detail | SeqNo | Form layout order |
| Grid | SeqNoGrid | Column order in list view |

> 📝 **Note:** Grid and detail layouts are independent. Changing one does not affect the other.

## SQL Patterns

### Query Current Field Layout

```sql
SELECT 
    f.ad_field_id,
    c.columnname,
    f.name,
    f.seqno,
    f.xposition,
    f.columnspan,
    f.seqnogrid
FROM ad_field f
JOIN ad_column c ON f.ad_column_id = c.ad_column_id
WHERE f.ad_tab_id = <tab_id>
  AND f.isactive = 'Y'
ORDER BY f.seqno;
```

### Update Field Position

```sql
-- Move field to new position
UPDATE ad_field 
SET seqno = <new_seqno>, 
    xposition = <x_pos>, 
    columnspan = <span>
WHERE ad_field_id = (
    SELECT f.ad_field_id 
    FROM ad_field f 
    JOIN ad_column c ON f.ad_column_id = c.ad_column_id 
    WHERE f.ad_tab_id = <tab_id> 
      AND c.columnname = '<column_name>'
);
```

## Common Lookups

**Find AD_Tab_ID:**
```sql
SELECT t.ad_tab_id, t.name, w.name as window_name
FROM ad_tab t 
JOIN ad_window w ON t.ad_window_id = w.ad_window_id
WHERE w.name ILIKE '%material receipt%'
  AND t.name ILIKE '%receipt line%';
```

**Find Field by Column Name:**
```sql
SELECT f.ad_field_id, c.columnname, f.name, f.seqno
FROM ad_field f
JOIN ad_column c ON f.ad_column_id = c.ad_column_id
WHERE f.ad_tab_id = <tab_id>
  AND c.columnname = 'ANS_PriceCost';
```

## Post-Changes

After modifying field layouts:
1. Reset cache: System Admin => Cache Reset
2. Log out and back in to see changes

## Reference

- See [idempiere-column-create-tool.md](idempiere-column-create-tool.md) for creating new columns and fields
- See [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md) for AD table patterns

Tags: #tool #idempiere #window #field-layout
