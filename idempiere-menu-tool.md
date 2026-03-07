---
name: idempiere-menu
description: Menu creation and management patterns for iDempiere including AD_Menu entries, tree node placement, and role visibility considerations
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-menu-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Menu Tool

The purpose of this document is to provide patterns for creating menu entries in iDempiere.

This is important because menu entries make processes discoverable and accessible to users without requiring window buttons or Info Window integration.

## Menu Architecture

Menus in iDempiere consist of two related tables:

```
AD_Menu (menu metadata)
    └── AD_TreeNodeMM (tree placement - where it appears in the menu hierarchy)
```

### Table Relationships

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| AD_Menu | Menu entry metadata (name, action, process link) | ad_menu_id, name, action, ad_process_id |
| AD_TreeNodeMM | Tree structure (parent-child relationships) | ad_tree_id, node_id, parent_id, seqno |

## Menu Entry Pattern

Creates a menu entry that appears in the main menu tree.

### SQL Template

```sql
-- ============================================================================
-- Create Menu Entry
-- ============================================================================

DO $$
DECLARE
    v_menu_id numeric;
    v_process_id numeric;
    v_tree_id numeric := 10;  -- Primary Menu tree (ID may vary by client)
BEGIN
    -- Get the process ID
    SELECT ad_process_id INTO v_process_id
    FROM ad_process WHERE value = 'ProcessValue';

    IF v_process_id IS NULL THEN
        RAISE EXCEPTION 'Process "ProcessValue" not found';
    END IF;

    -- Check if menu already exists
    SELECT ad_menu_id INTO v_menu_id
    FROM ad_menu WHERE name = 'Menu Display Name' AND action = 'P';

    IF v_menu_id IS NOT NULL THEN
        RAISE NOTICE 'Menu already exists (ID: %), skipping', v_menu_id;
        RETURN;
    END IF;

    -- Create the menu entry
    v_menu_id := nextval('ad_menu_sq');

    INSERT INTO ad_menu (
        ad_menu_id, ad_client_id, ad_org_id, isactive, created, createdby,
        updated, updatedby, name, description, issummary, issotrx, isreadonly,
        action, ad_process_id, ad_window_id, ad_workflow_id, ad_task_id,
        ad_form_id, ad_workbench_id, ad_infowindow_id, entitytype,
        iscentrallymaintained, ad_menu_uu
    ) VALUES (
        v_menu_id, 0, 0, 'Y', now(), 100,
        now(), 100,
        'Menu Display Name',
        'Description of what this menu does',
        'N',  -- issummary = No (not a folder)
        'Y',  -- issotrx = Yes (sales side, or 'N' for purchasing)
        'N',  -- isreadonly = No
        'P',  -- action = Process
        v_process_id,  -- ad_process_id
        NULL, -- ad_window_id
        NULL, -- ad_workflow_id
        NULL, -- ad_task_id
        NULL, -- ad_form_id
        NULL, -- ad_workbench_id
        NULL, -- ad_infowindow_id
        'U',  -- entitytype = User maintained
        'N',  -- iscentrallymaintained
        uuid_generate_v4()::varchar
    );

    RAISE NOTICE 'Created AD_Menu (ID: %)', v_menu_id;

    -- Add to menu tree
    IF EXISTS (
        SELECT 1 FROM ad_treenodeMM 
        WHERE ad_tree_id = v_tree_id AND node_id = v_menu_id
    ) THEN
        RAISE NOTICE 'Tree node already exists';
        RETURN;
    END IF;

    INSERT INTO ad_treenodeMM (
        ad_tree_id, node_id, ad_client_id, ad_org_id, isactive,
        created, createdby, updated, updatedby,
        parent_id, seqno, ad_treenodeMM_uu
    ) VALUES (
        v_tree_id,
        v_menu_id,
        0, 0, 'Y',
        now(), 100, now(), 100,
        0,     -- parent_id = 0 (root level) or specific parent menu ID
        999,   -- seqno = 999 (append at end)
        uuid_generate_v4()::varchar
    );

    RAISE NOTICE 'Added to menu tree %', v_tree_id;
END $$;
```

> 🔗 **Reference**: Verify table and column names match your iDempiere version. Use `\d ad_menu` and `\d ad_treenodeMM` in psql to check actual structure.

### Key Columns

**AD_Menu:**

| Column | Value | Notes |
|--------|-------|-------|
| action | 'P' | Process (use 'W' for Window, 'F' for Form, etc.) |
| ad_process_id | process ID | Links to AD_Process |
| issummary | 'N' | 'Y' for folder/menu group, 'N' for leaf item |
| issotrx | 'Y'/'N' | 'Y' for sales-side menus, 'N' for purchasing-side |

**AD_TreeNodeMM:**

| Column | Value | Notes |
|--------|-------|-------|
| ad_tree_id | 10 | Primary menu tree (verify in your environment) |
| parent_id | 0 or ID | 0 = root level, or specific parent menu ID |
| seqno | 999 | Display order (lower = higher in list) |

### Tree Placement Options

| Location | parent_id | Use Case |
|----------|-----------|----------|
| Root level | 0 | Standalone administrative tools |
| Under Sales | [Sales menu ID] | Sales-related processes |
| Under Purchasing | [Purchasing menu ID] | Purchasing-related processes |

To find parent menu IDs:

```sql
-- Find Sales Management menu
SELECT ad_menu_id, name FROM ad_menu WHERE name ILIKE '%sales%' AND issummary = 'Y';

-- List root-level menus
SELECT tnm.node_id, m.name 
FROM ad_treenodeMM tnm 
JOIN ad_menu m ON tnm.node_id = m.ad_menu_id 
WHERE tnm.ad_tree_id = 10 AND tnm.parent_id = 0 AND m.issummary = 'Y'
ORDER BY tnm.seqno;
```

## Role Considerations

Unlike windows and processes, menu visibility is controlled through the menu tree structure rather than explicit access tables. However, when creating menus for specific functional areas, consider which roles need visibility.

### Manual vs System Roles

iDempiere distinguishes between:

| Type | ismanual | Description |
|------|----------|-------------|
| System Roles | 'N' | Built-in roles (System Administrator, System API Access) |
| Manual Roles | 'Y' | User-created roles (Sales User, Warehouse Admin, etc.) |

When adding functional menus, typically target manual roles that represent actual users.

### Role Selection Pattern

For menus targeting specific business functions, identify relevant roles:

```sql
-- List manual roles for selection
SELECT ad_role_id, name, description 
FROM ad_role 
WHERE ismanual = 'Y' 
  AND isactive = 'Y' 
  AND ad_client_id IN (0, [YOUR_CLIENT_ID])
ORDER BY name;
```

> 📝 **Note**: Menu visibility is primarily controlled through tree structure. The manual vs system role distinction helps identify which roles represent actual business users vs system/automation accounts. Review your role list and determine which roles should have visibility to the new menu based on their functional responsibilities.

## Cache Reset Requirement

After creating menu entries, a cache reset is **required** for the menu to appear:

**Option 1: System Admin Menu**
- System Admin > Cache Reset

**Option 2: Application Server Restart**
- Restart the iDempiere application server

Without cache reset, the menu entry exists in the database but will not appear in the user interface.

## Complete Example

Storage Fee Allocation menu (from deploy script 20260306140000_storage_fee_allocation_menu.sql):

```sql
DO $$
DECLARE
    v_menu_id numeric;
    v_process_id numeric;
    v_tree_id numeric := 10;
BEGIN
    SELECT ad_process_id INTO v_process_id
    FROM ad_process WHERE value = 'StorageFeeAllocation';

    IF v_process_id IS NULL THEN
        RAISE EXCEPTION 'Process not found';
    END IF;

    SELECT ad_menu_id INTO v_menu_id
    FROM ad_menu WHERE name = 'Storage Fee Allocation' AND action = 'P';

    IF v_menu_id IS NOT NULL THEN
        RETURN;
    END IF;

    v_menu_id := nextval('ad_menu_sq');

    INSERT INTO ad_menu (
        ad_menu_id, ad_client_id, ad_org_id, isactive, created, createdby,
        updated, updatedby, name, description, issummary, issotrx, isreadonly,
        action, ad_process_id, ad_window_id, ad_workflow_id, ad_task_id,
        ad_form_id, ad_workbench_id, ad_infowindow_id, entitytype,
        iscentrallymaintained, ad_menu_uu
    ) VALUES (
        v_menu_id, 0, 0, 'Y', now(), 100, now(), 100,
        'Storage Fee Allocation',
        'Creates invoices for sales orders with unshipped reserved quantities',
        'N', 'Y', 'N', 'P', v_process_id,
        NULL, NULL, NULL, NULL, NULL, NULL, 'U', 'N', uuid_generate_v4()::varchar
    );

    INSERT INTO ad_treenodeMM (
        ad_tree_id, node_id, ad_client_id, ad_org_id, isactive,
        created, createdby, updated, updatedby,
        parent_id, seqno, ad_treenodeMM_uu
    ) VALUES (
        v_tree_id, v_menu_id, 0, 0, 'Y', now(), 100, now(), 100,
        0, 999, uuid_generate_v4()::varchar
    );
END $$;
```

## Related Documentation

- See [idempiere-process-tool.md](idempiere-process-tool.md) for creating the processes that menus link to
- See [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md) for general AD patterns

Tags: #tool #idempiere #menu #application-dictionary
