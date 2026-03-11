---
name: idempiere-cache-reset
description: Reset iDempiere application cache to reflect configuration changes without service restart
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-cache-reset-tool.md
  category: system-administration
  scope: idempiere
---

# iDempiere Cache Reset Tool

The purpose of this tool is to reset the iDempiere application cache after configuration changes.

This is important because the application caches frequently-accessed metadata (windows, menus, processes, etc.) for performance. After making changes to the application dictionary or deploying new components, the cache must be refreshed for changes to take effect.

## TOC

- [When to Reset Cache](#when-to-reset-cache)
- [UI Method](#ui-method)
- [Programmatic Method](#programmatic-method)
- [Alternative: Logout/Login](#alternative-logoutlogin)

## When to Reset Cache

Reset cache when you make changes to:

| Change Type | Cache Reset Required |
|-------------|---------------------|
| New menu entries | Yes |
| New model validators | Yes |
| New processes or reports | Yes |
| Window/tab/field modifications | Yes |
| Document type configurations | Sometimes |
| Business partner or product data | No |
| Transactional data | No |

> **📝 Note** - Cache reset is **not** required for data changes (invoices, orders, inventory). It is required for metadata changes that affect how the application behaves.

## UI Method

Use the UI method when:
- You have System Administrator access
- You are making one-time changes
- You can interact with the web interface

**Steps:**

1. Log in with **System Administrator** role
2. Navigate to **System Admin** > **General Rules** > **Cache Reset**
3. Click the **Reset Cache** button
4. Wait for the "Cache Reset" confirmation message

## Programmatic Method

Use the programmatic method when:
- You are deploying via scripts
- You need to reset cache as part of automation
- UI access is not available or practical

### Prerequisites

Before running cache reset programmatically, ensure:

1. **System API Access role exists**
   - See `20260107235400_system_api_role.sql` deployment script

2. **Process access granted**
   - Process ID: `205` (Cache Reset)
   - Must have `AD_Process_Access` record for System API Access role

### REST API Pattern

```bash
#!/usr/bin/env bash

API_URL="http://localhost:8080/api/v1"
PROCESS_SLUG="cache-reset"

# Step 1: Get System API Access role ID
ROLE_ID=$(psqli -t -A -c "SELECT ad_role_id FROM adempiere.ad_role WHERE name = 'System API Access';")

# Step 2: Grant process access (idempotent)
psqli -c "
INSERT INTO adempiere.ad_process_access (
    ad_process_access_uu, ad_client_id, ad_org_id, ad_process_id, ad_role_id,
    isactive, isreadwrite, created, createdby, updated, updatedby
)
SELECT
    generate_uuid(), 0, 0, 205, ${ROLE_ID},
    'Y', 'Y', now(), 100, now(), 100
WHERE NOT EXISTS (
    SELECT 1 FROM adempiere.ad_process_access
    WHERE ad_process_id = 205 AND ad_role_id = ${ROLE_ID}
);
"

# Step 3: Authenticate as SuperUser
AUTH_RESPONSE=$(curl -s -X POST "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -d '{"userName":"SuperUser","password":"System"}')

TOKEN=$(echo "$AUTH_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)

# Step 4: Select System client with System API Access role
SESSION_RESPONSE=$(curl -s -X PUT "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d "{\"clientId\":0,\"roleId\":${ROLE_ID},\"organizationId\":0,\"warehouseId\":0}")

SESSION_TOKEN=$(echo "$SESSION_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)

# Step 5: Run cache reset
curl -s -X POST "${API_URL}/processes/${PROCESS_SLUG}" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{}'
```

### Success Response

```json
{
  "AD_PInstance_ID": 1000079,
  "process": "cache-reset",
  "summary": "Cache Reset",
  "isError": false,
  "nodeId": "8ebd06c6-6870-4c36-bf38-40d88b8267a7"
}
```

### Verification

After cache reset, verify the change took effect:

1. **For menu entries:** Check if new menu appears
2. **For model validators:** Create a test record that triggers the validator
3. **For processes:** Verify process appears in action menu

## Alternative: Logout/Login

Some AD changes take effect without explicit cache reset:

- Log out of iDempiere
- Log back in with same or different user
- Browser refresh (F5 or Ctrl+R) may help for UI-only changes

> **💡 Tip** - Try logout/login first. If the change still doesn't appear, perform a full cache reset.

## Troubleshooting

### Cache Reset Button Grayed Out

- Ensure you are logged in as **System Administrator** role
- Check role has access to Cache Reset process (ID 205)

### Changes Still Not Appearing After Reset

1. Verify the change was actually saved to database
2. Check you are looking in the correct client/org context
3. Try full browser refresh (Ctrl+F5 to clear browser cache)
4. Restart iDempiere service if cache reset fails repeatedly

### REST API Returns 401 Unauthorized

- Verify System API Access role exists
- Check process access is granted to the role
- Ensure SuperUser credentials are correct
- Verify iDempiere REST API plugin is active

Tags: #tool #cache #system-admin #idempiere
