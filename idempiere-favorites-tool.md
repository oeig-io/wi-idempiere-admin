---
name: idempiere-favorites
description: Programmatic management of iDempiere user favorites including menu items, folders, and hierarchical organization via AD_Tree_Favorite and AD_Tree_Favorite_Node tables
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-favorites-tool.md
  category: user-preferences
  scope: idempiere
---

# iDempiere Favorites Tool

The purpose of this document is to enable programmatic management of iDempiere user favorites.

This is important because favorites help users quickly access frequently used windows, and programmatic management ensures consistent initial setups across deployments while allowing custom folder organization.

## Favorites Database Structure

```
AD_Tree_Favorite (one per user)
    └── AD_Tree_Favorite_Node (menu items and folders)
```

> 🔗 **Reference** - Favorites use their own node table (`AD_Tree_Favorite_Node`). For the general tree model shared with the organization and menu trees, see the tree maintenance tool.

## AD_Tree_Favorite

One header record per user. All favorites for a user link to this record via `ad_tree_favorite_id`.

**Key Columns:**

| Column | Purpose |
|--------|---------|
| ad_tree_favorite_id | Primary key |
| ad_user_id | User who owns these favorites |
| ad_client_id | Always 0 (System) for the header |
| ad_org_id | Always 0 |

**Get User's Favorite Record:**

```sql
SELECT ad_tree_favorite_id, ad_user_id
FROM ad_tree_favorite
WHERE ad_user_id = <user_id>;
```

**Create New Favorite Record:**

```sql
INSERT INTO ad_tree_favorite (
    ad_tree_favorite_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, ad_user_id, ad_tree_favorite_uu
)
VALUES (
    nextval('ad_tree_favorite_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    <user_id>, uuid_generate_v4()
);
```

## AD_Tree_Favorite_Node

Individual menu items and folders. Child nodes reference parent folders via `parent_id`.

**Node Types:**

| Type | issummary | ad_menu_id | name | parent_id |
|------|-----------|------------|------|-----------|
| Folder | 'Y' | NULL | Folder name | NULL or parent folder |
| Menu Item | 'N' | <menu_id> | Empty or custom | NULL or parent folder |

**Key Columns:**

| Column | Purpose |
|--------|---------|
| ad_tree_favorite_node_id | Primary key |
| ad_tree_favorite_id | Links to user's header record |
| ad_menu_id | Menu item (NULL for folders) |
| name | Folder name (empty for menu items) |
| issummary | 'Y' = folder, 'N' = menu item |
| parent_id | Parent folder node ID (NULL = root level) |
| seqno | Display order within parent (0 = alphabetical) |
| ad_client_id | Client context ([CLIENT_ID]) |

> **📝 Note:** Items display alphabetically when `seqno = 0`. Users can drag-and-drop reorder via WebUI.

## Creating Flat Favorites (Root Level)

Create menu items without folders at the root level.

**Pattern:**

```sql
DO $$
DECLARE
    v_favorite_id INTEGER;
    v_client_id INTEGER;
    v_menu_ids INTEGER[] := ARRAY[<menu_id_1>, <menu_id_2>];
    v_menu_id INTEGER;
BEGIN
    -- Get user's favorite record
    SELECT ad_tree_favorite_id INTO v_favorite_id
    FROM ad_tree_favorite
    WHERE ad_user_id = <user_id>;

    -- Get ACME client ID
    SELECT ad_client_id INTO v_client_id
    FROM ad_client WHERE name = 'ACME';

    -- Create favorites
    FOREACH v_menu_id IN ARRAY v_menu_ids
    LOOP
        INSERT INTO ad_tree_favorite_node (
            ad_tree_favorite_node_id, ad_tree_favorite_id, ad_client_id, ad_org_id,
            ad_menu_id, isactive, iscollapsible, issummary, isfavourite,
            created, createdby, updated, updatedby, name, seqno, ad_tree_favorite_node_uu
        )
        VALUES (
            nextval('ad_tree_favorite_node_sq'),
            v_favorite_id, v_client_id, 0,
            v_menu_id, 'Y', 'Y', 'N', 'Y',
            now(), 100, now(), 100, '', 0, uuid_generate_v4()
        )
        ON CONFLICT DO NOTHING;
    END LOOP;
END $$;
```

## Creating Folder Structures

Create hierarchical organization with folders containing nested items.

**Create Folder:**

```sql
-- Insert folder first, capture its ID
INSERT INTO ad_tree_favorite_node (
    ad_tree_favorite_node_id, ad_tree_favorite_id, ad_client_id, ad_org_id,
    ad_menu_id, isactive, iscollapsible, issummary, isfavourite,
    created, createdby, updated, updatedby, name, seqno, ad_tree_favorite_node_uu
)
VALUES (
    nextval('ad_tree_favorite_node_sq'),
    <favorite_id>, <client_id>, 0,
    NULL, 'Y', 'Y', 'Y', 'Y',  -- issummary = 'Y' for folder
    now(), 100, now(), 100, 'Folder Name', 0, uuid_generate_v4()
)
RETURNING ad_tree_favorite_node_id;
```

**Add Item to Folder:**

```sql
INSERT INTO ad_tree_favorite_node (
    ad_tree_favorite_node_id, ad_tree_favorite_id, ad_client_id, ad_org_id,
    ad_menu_id, isactive, iscollapsible, issummary, isfavourite,
    created, createdby, updated, updatedby, name, seqno, parent_id, ad_tree_favorite_node_uu
)
VALUES (
    nextval('ad_tree_favorite_node_sq'),
    <favorite_id>, <client_id>, 0,
    <menu_id>, 'Y', 'Y', 'N', 'Y',  -- issummary = 'N' for menu item
    now(), 100, now(), 100, '', <seqno>, <folder_node_id>, uuid_generate_v4()
);
```

**Complete Example - Folder with Multiple Items:**

```sql
DO $$
DECLARE
    v_favorite_id INTEGER;
    v_client_id INTEGER;
    v_folder_id INTEGER;
BEGIN
    -- Get user's favorite record
    SELECT ad_tree_favorite_id INTO v_favorite_id
    FROM ad_tree_favorite WHERE ad_user_id = 100;

    -- Get ACME client ID
    SELECT ad_client_id INTO v_client_id
    FROM ad_client WHERE name = 'ACME';

    -- Create folder
    INSERT INTO ad_tree_favorite_node (
        ad_tree_favorite_node_id, ad_tree_favorite_id, ad_client_id, ad_org_id,
        ad_menu_id, isactive, iscollapsible, issummary, isfavourite,
        created, createdby, updated, updatedby, name, seqno, ad_tree_favorite_node_uu
    )
    VALUES (
        nextval('ad_tree_favorite_node_sq'), v_favorite_id, v_client_id, 0,
        NULL, 'Y', 'Y', 'Y', 'Y',
        now(), 100, now(), 100, 'Transactions', 0, uuid_generate_v4()
    )
    RETURNING ad_tree_favorite_node_id INTO v_folder_id;

    -- Add Sales Order to folder
    INSERT INTO ad_tree_favorite_node (
        ad_tree_favorite_node_id, ad_tree_favorite_id, ad_client_id, ad_org_id,
        ad_menu_id, isactive, iscollapsible, issummary, isfavourite,
        created, createdby, updated, updatedby, name, seqno, parent_id, ad_tree_favorite_node_uu
    )
    VALUES (
        nextval('ad_tree_favorite_node_sq'), v_favorite_id, v_client_id, 0,
        129, 'Y', 'Y', 'N', 'Y',
        now(), 100, now(), 100, '', 10, v_folder_id, uuid_generate_v4()
    );

    -- Add Purchase Order to folder
    INSERT INTO ad_tree_favorite_node (
        ad_tree_favorite_node_id, ad_tree_favorite_id, ad_client_id, ad_org_id,
        ad_menu_id, isactive, iscollapsible, issummary, isfavourite,
        created, createdby, updated, updatedby, name, seqno, parent_id, ad_tree_favorite_node_uu
    )
    VALUES (
        nextval('ad_tree_favorite_node_sq'), v_favorite_id, v_client_id, 0,
        205, 'Y', 'Y', 'N', 'Y',
        now(), 100, now(), 100, '', 20, v_folder_id, uuid_generate_v4()
    );
END $$;
```

## Role-Based Bulk Creation

Query users by role and apply favorites programmatically. See the deploy script `20260228210000_ans_role_favorites.sql` in `idempiere-golive-deploy/deploy/` for a complete working example.

**Pattern:**
- Query users from `ad_user_roles` by role ID
- Create `ad_tree_favorite` header if missing
- Insert favorite nodes for each user
- Use `ON CONFLICT DO NOTHING` to avoid duplicates

## Common Menu IDs

Frequently used favorites for ACME operations:

| Menu ID | Name | Purpose |
|---------|------|---------|
| 110 | Business Partner | Customer/vendor management |
| 126 | Product | Product master data |
| 129 | Sales Order | Sales order entry |
| 205 | Purchase Order | Purchase order entry |
| 204 | Material Receipt | Goods receipt |
| 180 | Shipment (Customer) | Outbound shipments |
| 206 | Purchase Invoice | Vendor billing |
| 178 | Sales Invoice | Customer billing |
| 235 | Payment and Receipt | Payment processing |

## Querying Favorites

**List All User's Favorites:**

```sql
SELECT 
    tfn.ad_tree_favorite_node_id,
    COALESCE(m.name, tfn.name) as item_name,
    tfn.issummary,
    tfn.parent_id,
    tfn.ad_client_id,
    tfn.seqno,
    CASE 
        WHEN p.name IS NOT NULL THEN p.name
        ELSE 'Root'
    END as parent_folder
FROM ad_tree_favorite_node tfn
LEFT JOIN ad_menu m ON tfn.ad_menu_id = m.ad_menu_id
LEFT JOIN ad_tree_favorite_node p ON tfn.parent_id = p.ad_tree_favorite_node_id
WHERE tfn.ad_tree_favorite_id = (
    SELECT ad_tree_favorite_id FROM ad_tree_favorite WHERE ad_user_id = 100
)
ORDER BY tfn.parent_id NULLS FIRST, tfn.seqno;
```

**Find Favorites by Client:**

```sql
SELECT tfn.ad_tree_favorite_node_id, m.name, tfn.ad_client_id, c.name as client_name
FROM ad_tree_favorite_node tfn
JOIN ad_menu m ON tfn.ad_menu_id = m.ad_menu_id
JOIN ad_client c ON tfn.ad_client_id = c.ad_client_id
WHERE tfn.ad_tree_favorite_id = (
    SELECT ad_tree_favorite_id FROM ad_tree_favorite WHERE ad_user_id = 100
)
AND tfn.ad_client_id = [CLIENT_ID]  -- ACME
ORDER BY tfn.seqno;
```

**Find Duplicate Menu IDs:**

```sql
SELECT ad_menu_id, COUNT(*) as cnt
FROM ad_tree_favorite_node
WHERE ad_tree_favorite_id = (
    SELECT ad_tree_favorite_id FROM ad_tree_favorite WHERE ad_user_id = 100
)
AND ad_client_id = 1000000
GROUP BY ad_menu_id
HAVING COUNT(*) > 1;
```

## Deleting Favorites

**Delete by Client:**

```sql
DELETE FROM ad_tree_favorite_node
WHERE ad_tree_favorite_id = (
    SELECT ad_tree_favorite_id FROM ad_tree_favorite WHERE ad_user_id = 100
)
AND ad_client_id = <client_id>;
```

**Delete Specific Menu:**

```sql
DELETE FROM ad_tree_favorite_node
WHERE ad_tree_favorite_id = (
    SELECT ad_tree_favorite_id FROM ad_tree_favorite WHERE ad_user_id = 100
)
AND ad_menu_id = <menu_id>
AND ad_client_id = <client_id>;
```

**Delete Folder and Contents:**

```sql
-- Delete children first
DELETE FROM ad_tree_favorite_node
WHERE parent_id = <folder_node_id>;

-- Delete folder
DELETE FROM ad_tree_favorite_node
WHERE ad_tree_favorite_node_id = <folder_node_id>;
```

## Post-Changes

Favorites changes take effect immediately without cache reset. Users may need to refresh the browser or log out/in to see updates in the UI.

Tags: #tool #idempiere #favorites #user-preferences #menu
