---
name: idempiere-document-status
description: Configure home page Activities widget indicators using PA_DocumentStatus for counting records and linking to windows, forms, processes, or info windows
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-document-status-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Document Status Tool

## TOC

- [Overview](#overview)
- [Architecture](#architecture)
- [PA_DocumentStatus](#pa_documentstatus)
- [Click Action](#click-action)
- [WhereClause](#whereclause)
- [Context Variables](#context-variables)
- [Access Control](#access-control)
- [Visual Customization](#visual-customization)
- [Field Groups](#field-groups)
- [Complete Example](#complete-example)
- [Verification](#verification)
- [References](#references)

## Overview

The purpose of this tool document is to describe how to create and configure Document Status indicators that appear in the home page Activities widget.

This is important because Document Status indicators give users at-a-glance counts of actionable items (open orders, pending approvals, unprocessed documents) with one-click navigation to the relevant records.

The Activities widget is backed by PA_DashboardContent record 200003 (`/zul/docstatus.zul`). Each row in the widget is a PA_DocumentStatus record that counts matching records in a target table and displays the result. When clicked, it opens a window, form, process, or info window filtered to the matching records.

## Architecture

| Component | Table | Purpose |
|-----------|-------|---------|
| Indicator | PA_DocumentStatus | Defines what to count and where to navigate |
| Access | PA_DocumentStatusAccess | Controls which roles/users see each indicator |
| Dashboard | PA_DashboardContent | Container widget on home page (already exists) |

The evaluation flow:
1. `MDocumentStatus.getDocumentStatusIndicators()` loads all active indicators for the user's client
2. Each indicator is filtered by access control (PA_DocumentStatusAccess)
3. Each indicator is filtered by role access to the target window/form/process/info window
4. `MDocumentStatus.evaluate()` runs `SELECT COUNT(*) FROM <table> WHERE <whereclause>` with role-based access SQL appended
5. The count is displayed; if `IsHideWhenZero='Y'` and count is 0, the row is hidden

## PA_DocumentStatus

| Column | Type | Purpose |
|--------|------|---------|
| Name | String | Display label shown in widget |
| Description | String | Tooltip text |
| Help | String | Extended help; shown as message dialog when no Window/Form/Process/InfoWindow is set |
| AD_Table_ID | Table Direct | Table to query for COUNT(*) |
| WhereClause | Text | SQL WHERE clause with optional context variables |
| AD_Window_ID | Table Direct | Window to open on click (highest priority) |
| AD_Form_ID | Table Direct | Form to open on click |
| AD_Process_ID | Table Direct | Process to open on click |
| AD_InfoWindow_ID | Table Direct | Info window to open on click |
| SeqNo | Integer | Sort order within widget |
| IsHideWhenZero | Yes/No | Hide row when count is 0 |
| AD_FieldGroup_ID | Table Direct | Visual grouping header |
| Name_PrintColor_ID | Table Direct | Color for name label |
| Name_PrintColorZero_ID | Table Direct | Color for name label when count is 0 |
| Name_PrintFont_ID | Table Direct | Font for name label |
| Number_PrintColor_ID | Table Direct | Color for count number |
| Number_PrintFont_ID | Table Direct | Font for count number |
| C_Project_ID | Table Direct | Optional project filter added to WHERE clause |
| AD_Org_ID | Table Direct | Optional org filter added to WHERE clause (when > 0) |

> ⚠️ **Warning** - Only one of AD_Window_ID, AD_Form_ID, AD_Process_ID, or AD_InfoWindow_ID can be set. The beforeSave logic clears the others based on priority: Window > Form > Process > InfoWindow.

```sql
INSERT INTO pa_documentstatus (
    pa_documentstatus_id, ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby,
    name, description, ad_table_id, whereclause,
    ad_window_id, seqno, ishidewhenzero, entitytype,
    pa_documentstatus_uu
) VALUES (
    nextval('pa_documentstatus_sq'), 11, 0, 'Y',
    now(), 100, now(), 100,
    'Draft Sales Orders',
    'Sales orders in draft status',
    259,  -- C_Order
    'IsSOTrx=''Y'' AND DocStatus=''DR'' AND CreatedBy=@#AD_User_ID@',
    143,  -- Sales Order window
    100,
    'N',
    'U',
    uuid_generate_v4()
);
```

> 📝 **Note** - If SeqNo is 0 or not set, beforeSave auto-assigns the next value as `MAX(SeqNo) + 10`.

## Click Action

When a user clicks an indicator row, the action depends on which ID is set:

| Priority | Field | Behavior |
|----------|-------|----------|
| 1 | AD_Window_ID | Opens window with MQuery filtered to matching records |
| 2 | AD_Form_ID | Opens the form |
| 3 | AD_Process_ID | Opens the process dialog |
| 4 | AD_InfoWindow_ID | Opens the info window |
| None | (all empty) | Shows Help text in a message dialog |

The user's role must have access to the target window/form/process/info window, otherwise the indicator is not shown.

## WhereClause

The WhereClause is appended to a generated query: `SELECT COUNT(*) FROM <tablename> WHERE <tablename>.AD_Client_ID=? [AND C_Project_ID=?] [AND AD_Org_ID=?] AND (<WhereClause>)`.

Role-based access SQL is also appended automatically via `MRole.addAccessSQL()`.

Rules:
- Reference the table name in column references when needed for clarity (e.g., `R_Request.SalesRep_ID`)
- Use subqueries for complex logic (e.g., `EXISTS (SELECT ...)`)
- Single quotes in SQL must be doubled: `DocStatus=''DR''`

## Context Variables

The WhereClause supports iDempiere context variables that are resolved at runtime:

| Variable | Description |
|----------|-------------|
| `@#AD_User_ID@` | Current login user ID |
| `@#AD_Role_ID@` | Current login role ID |
| `@#AD_Client_ID@` | Current login client ID |
| `@#AD_Org_ID@` | Current login org ID |
| `SysDate` | Current database date (PostgreSQL `now()`) |

These are login-context variables (prefixed with `#`), not window-context variables.

## Access Control

PA_DocumentStatusAccess controls visibility per role and/or user.

| Column | Purpose |
|--------|---------|
| PA_DocumentStatus_ID | Links to the indicator |
| AD_Role_ID | Role that can see this indicator (0 = any role) |
| AD_User_ID | User that can see this indicator (0 = any user) |

**Access logic:**
- If NO access records exist for an indicator → visible to all roles/users
- If access records exist → user must match at least one record by:
  - Exact role AND exact user match, OR
  - Exact role match with user = 0 (any user in that role), OR
  - Role = 0 with exact user match (user in any role)

```sql
-- Restrict indicator to a specific role
INSERT INTO pa_documentstatusaccess (
    pa_documentstatusaccess_id, ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby,
    pa_documentstatus_id, ad_role_id, ad_user_id,
    pa_documentstatusaccess_uu
) VALUES (
    nextval('pa_documentstatusaccess_sq'), 11, 0, 'Y',
    now(), 100, now(), 100,
    (SELECT pa_documentstatus_id FROM pa_documentstatus WHERE name = 'Draft Sales Orders' AND ad_client_id = 11),
    (SELECT ad_role_id FROM ad_role WHERE name = 'ACME Admin' AND ad_client_id = 11),
    0,  -- Any user with this role
    uuid_generate_v4()
);
```

## Visual Customization

Fonts and colors are set via Print Font and Print Color records. Look up existing IDs:

```sql
-- Available print colors
SELECT ad_printcolor_id, name FROM ad_printcolor WHERE isactive = 'Y' ORDER BY name;

-- Available print fonts
SELECT ad_printfont_id, name FROM ad_printfont WHERE isactive = 'Y' ORDER BY name;
```

The `Name_PrintColorZero_ID` field provides an alternate color for the name label when the count is zero. This is useful for indicators where zero is the desired state (e.g., "Default Passwords" shows green when count is 0).

## Field Groups

Use `AD_FieldGroup_ID` to visually group related indicators under a collapsible header. All indicators sharing the same field group appear together, ordered by SeqNo within the group.

Indicators are sorted by `COALESCE(AD_FieldGroup_ID, 0), SeqNo`. Ungrouped indicators (field group = null) appear first.

```sql
-- Find existing field groups
SELECT ad_fieldgroup_id, name FROM ad_fieldgroup WHERE isactive = 'Y' ORDER BY name;
```

## Complete Example

This example creates an indicator that shows the count of draft purchase orders for the current user, visible only to a specific role.

```sql
DO $$
DECLARE
    v_docstatus_id numeric;
BEGIN
    -- Step 1: Create the indicator
    INSERT INTO pa_documentstatus (
        pa_documentstatus_id, ad_client_id, ad_org_id, isactive,
        created, createdby, updated, updatedby,
        name, description, ad_table_id, whereclause,
        ad_window_id, seqno, ishidewhenzero, entitytype,
        pa_documentstatus_uu
    ) VALUES (
        nextval('pa_documentstatus_sq'), 11, 0, 'Y',
        now(), 100, now(), 100,
        'My Draft POs',
        'Purchase orders I created that are still in draft',
        259,  -- C_Order
        'IsSOTrx=''N'' AND DocStatus=''DR'' AND CreatedBy=@#AD_User_ID@',
        181,  -- Purchase Order window
        200,
        'Y',  -- Hide when zero
        'U',
        uuid_generate_v4()
    ) RETURNING pa_documentstatus_id INTO v_docstatus_id;

    -- Step 2: Restrict to purchasing role (optional)
    INSERT INTO pa_documentstatusaccess (
        pa_documentstatusaccess_id, ad_client_id, ad_org_id, isactive,
        created, createdby, updated, updatedby,
        pa_documentstatus_id, ad_role_id, ad_user_id,
        pa_documentstatusaccess_uu
    ) VALUES (
        nextval('pa_documentstatusaccess_sq'), 11, 0, 'Y',
        now(), 100, now(), 100,
        v_docstatus_id,
        (SELECT ad_role_id FROM ad_role WHERE name = 'ACME Purchasing' AND ad_client_id = 11),
        0,
        uuid_generate_v4()
    );
END $$;
```

## Verification

After creating a Document Status indicator:

1. Reset cache (see idempiere-cache-reset-tool.md)
2. Log out and log back in (or refresh the dashboard)
3. The Activities widget should show the new indicator with its count
4. Click the indicator to verify it opens the correct window/form/process with filtered results

```sql
-- Verify indicator exists
SELECT pa_documentstatus_id, name, seqno, ishidewhenzero, ad_table_id, ad_window_id
FROM pa_documentstatus
WHERE ad_client_id IN (0, 11)
AND isactive = 'Y'
ORDER BY COALESCE(ad_fieldgroup_id, 0), seqno;

-- Verify access records
SELECT ds.name, dsa.ad_role_id, r.name as role_name, dsa.ad_user_id
FROM pa_documentstatusaccess dsa
JOIN pa_documentstatus ds ON dsa.pa_documentstatus_id = ds.pa_documentstatus_id
LEFT JOIN ad_role r ON dsa.ad_role_id = r.ad_role_id
ORDER BY ds.name;

-- Test the count query manually (replace context variables)
SELECT COUNT(*) FROM c_order
WHERE c_order.ad_client_id = 11
AND (IsSOTrx='N' AND DocStatus='DR' AND CreatedBy=100);
```

## References

- Document Status window: ERP => Application Dictionary => Document Status
- Source: `org.compiere.model.MDocumentStatus` (evaluation and access logic)
- Source: `org.adempiere.webui.apps.graph.WDocumentStatusIndicator` (UI rendering and click handling)
- Source: `org.adempiere.webui.apps.graph.WDocumentStatusPanel` (panel layout and refresh)
- Source: `org.adempiere.webui.dashboard.DPDocumentStatus` (dashboard integration)

Tags: #tool #idempiere #document-status #activities #dashboard #home-page
