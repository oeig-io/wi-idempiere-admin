---
name: idempiere-tree-maintenance
description: Maintain iDempiere hierarchy trees (organization, and other element trees) via AD_Tree and the tree-node tables, including node placement, summary flags, and idempotent deploy patterns
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-tree-maintenance-tool.md
  category: configuration
  scope: idempiere
---

# iDempiere Tree Maintenance Tool

The purpose of this document is to enable programmatic maintenance of iDempiere hierarchy trees — most commonly the Organization tree.

This is important because hierarchies (consolidation parents, summary nodes, display order) drive reporting and navigation, yet they live in `AD_Tree*` tables that no single window edits cleanly. Deploy scripts must place nodes deterministically and idempotently so a fresh deploy reproduces the intended structure.

## Tree Model

Every tree has one header (`AD_Tree`) plus node rows in a tree-type-specific node table.

```
AD_Tree (one header per tree, identified by treetype + ad_client_id)
    └── AD_TreeNode<xx> (one row per element: node_id, parent_id, seqno)
```

| Concept | Detail |
|---------|--------|
| `AD_Tree.treetype` | Identifies the tree (e.g., `OO` organization, `MM` menu, `BP` business partner, `PR` product) |
| `node_id` | The element's primary key (e.g., `ad_org_id` for an org tree) |
| `parent_id` | `node_id` of the parent; `0` = tree root |
| `seqno` | Display order among siblings |

> 📝 **Note** - Find a client's tree by `treetype` + `ad_client_id`, never by hardcoded `ad_tree_id` (it differs per environment).

## Node Table by Tree Type

The node table depends on `treetype`. The generic `AD_TreeNode` serves the organization tree and most element trees; menu, business partner, and product have dedicated tables:

| treetype | Tree | Node table |
|----------|------|------------|
| `OO` | Organization | `ad_treenode` |
| `MM` | Menu | `ad_treenodemm` |
| `BP` | Business Partner | `ad_treenodebp` |
| `PR` | Product | `ad_treenodepr` |
| (user/favorites) | User favorites | `ad_tree_favorite_node` |

> 🔗 **Reference** - For menu placement (`AD_TreeNodeMM`), see the menu tool. For user favorites (`AD_Tree_Favorite_Node`), see the favorites tool. This document focuses on the Organization tree (`AD_TreeNode`, treetype `OO`); the patterns generalize to any element tree on the generic `AD_TreeNode`.

## Organization Tree Pattern

When an organization is created (Initial Client Setup, or `AD_Org` via REST API), iDempiere automatically adds it to the client's `OO` tree as a flat node directly under the root. Maintenance means re-pointing `parent_id` and `seqno` to build the intended hierarchy — the node already exists.

**Build the hierarchy (idempotent UPDATE, defensive INSERT):**

```sql
DO $$
DECLARE
    v_client_id numeric;
    v_tree_id   numeric;
    v_org_id    numeric;
    v_parent_id numeric;
    rec record;
    v_updated int;
BEGIN
    SELECT ad_client_id INTO v_client_id
    FROM ad_client WHERE name = '${CLIENT_NAME}';

    SELECT ad_tree_id INTO v_tree_id
    FROM ad_tree WHERE treetype = 'OO' AND ad_client_id = v_client_id;

    -- (org value, parent org value [NULL = root], seqno)
    FOR rec IN
        SELECT * FROM (VALUES
            ('oeig-cons',   NULL,        0),
            ('oeig-serv',   'oeig-cons', 0),
            ('bearly-cons', 'oeig-cons', 1)
        ) AS t(org_value, parent_value, seqno)
    LOOP
        SELECT ad_org_id INTO v_org_id
        FROM ad_org WHERE ad_client_id = v_client_id AND value = rec.org_value;

        IF rec.parent_value IS NULL THEN
            v_parent_id := 0;  -- tree root
        ELSE
            SELECT ad_org_id INTO v_parent_id
            FROM ad_org WHERE ad_client_id = v_client_id AND value = rec.parent_value;
        END IF;

        UPDATE ad_treenode
        SET parent_id = v_parent_id, seqno = rec.seqno, updated = now(), updatedby = 100
        WHERE ad_tree_id = v_tree_id AND node_id = v_org_id;
        GET DIAGNOSTICS v_updated = ROW_COUNT;

        IF v_updated = 0 THEN
            INSERT INTO ad_treenode (
                ad_tree_id, node_id, ad_client_id, ad_org_id, isactive,
                created, createdby, updated, updatedby,
                parent_id, seqno, ad_treenode_uu
            ) VALUES (
                v_tree_id, v_org_id, v_client_id, 0, 'Y',
                now(), 100, now(), 100,
                v_parent_id, rec.seqno, gen_random_uuid()
            );
        END IF;
    END LOOP;
END $$;
```

## Summary Organizations

A consolidation parent must be a Summary organization (non-postable). `deploy.properties` cannot express `IsSummary`, so flag summary orgs explicitly in a deploy script:

```sql
UPDATE ad_org
SET issummary = 'Y', updated = now(), updatedby = 100
WHERE ad_client_id = (SELECT ad_client_id FROM ad_client WHERE name = '${CLIENT_NAME}')
  AND value = ANY (ARRAY['oeig-cons', 'bearly-cons'])
  AND issummary <> 'Y';
```

> 🔗 **Reference** - Working examples: `idempiere-golive-deploy/deploy/` contains `..._org_summary_flags.sql` (summary flags) and `..._org_tree_structure.sql` (org tree hierarchy).

## Conventions

- **Update in place; do not delete.** Every active element must keep exactly one node. Re-pointing `parent_id`/`seqno` fully expresses the target state — deleting and recreating risks an un-parented element if a later step fails. Deleting an element cascades to its node via FK, so there are no orphans to clean up.
- **Resolve everything dynamically.** Look up `ad_client_id`, `ad_tree_id`, and each `node_id` by name/value; never hardcode environment-specific IDs.
- **`parent_id = 0` is the root**, not "all organizations". It is a tree-structure constant.
- **Idempotent.** Re-running re-asserts the same parent/seqno.
- **Cache.** Tree changes are cached; users log out/in to see them in the UI. Do not run a cache reset without approval.

> 🔗 **Reference** - For portable SQL conventions (variable substitution, dynamic lookups, `.sql` vs `.sh`), see the deploy SQL tool.

Tags: #tool #idempiere #tree #organization #hierarchy #deploy
