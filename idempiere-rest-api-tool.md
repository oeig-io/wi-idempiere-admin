---
name: idempiere-rest-api
description: REST API patterns for authentication, CRUD operations, nested record creation, process execution, and query filtering in iDempiere
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-rest-api-tool.md
  category: integration
  scope: idempiere
---

# iDempiere REST API Tool

The purpose of this document is to provide working patterns for the iDempiere REST API.

## Authentication

Two-step process: get token, then select context. The credentials and role depend on which client you need.

### Client-Level Calls

For operations on tenant data (orders, invoices, business partners):

```bash
API_URL="http://localhost:8080/api/v1"

# Step 1: Authenticate as tenant admin
AUTH_RESPONSE=$(curl -s -X POST "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -d "{\"userName\":\"${ADMIN_USER}\",\"password\":\"${ADMIN_PASSWORD}\"}")

TOKEN=$(echo "$AUTH_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)

# Step 2: Select client context
SESSION_RESPONSE=$(curl -s -X PUT "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d "{\"clientId\":${CLIENT_ID},\"roleId\":${ROLE_ID},\"organizationId\":${ORG_ID},\"warehouseId\":${WH_ID}}")

SESSION_TOKEN=$(echo "$SESSION_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)
```

### System-Level Calls

For operations on system/dictionary data (PackOut, native sequences, application dictionary). Three things differ from client calls:

1. **User**: `SuperUser` / `System`
2. **Role**: `System API Access` (not `System Administrator` — role ID 0 returns `Invalid roleId`)
3. **Process access**: The System API Access role needs explicit `ad_process_access` grants for each process

```bash
API_URL="http://localhost:8080/api/v1"
ROLE_ID=$(psqli -t -A -c "SELECT ad_role_id FROM ad_role WHERE name = 'System API Access';")

# Step 1: Authenticate as SuperUser
AUTH_RESPONSE=$(curl -s -X POST "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -d '{"userName":"SuperUser","password":"System"}')

TOKEN=$(echo "$AUTH_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)

# Step 2: Select System client
SESSION_RESPONSE=$(curl -s -X PUT "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d "{\"clientId\":0,\"roleId\":${ROLE_ID},\"organizationId\":0,\"warehouseId\":0}")

SESSION_TOKEN=$(echo "$SESSION_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)
```

### Granting Process Access to System API Access Role

Before calling a process via system-level API, grant access (idempotent):

```sql
INSERT INTO ad_process_access (ad_process_id, ad_role_id, ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby, isreadwrite, ad_process_access_uu)
SELECT v_process_id, ad_role_id, 0, 0, 'Y', now(), 100, now(), 100, 'Y', uuid_generate_v4()::varchar
FROM ad_role WHERE name = 'System API Access'
AND NOT EXISTS (
    SELECT 1 FROM ad_process_access pa
    JOIN ad_role r ON pa.ad_role_id = r.ad_role_id
    WHERE pa.ad_process_id = v_process_id AND r.name = 'System API Access'
);
```

## Common ID Lookups

Use `psqli -t -A -c` for single-value queries (trimmed, unaligned).

```bash
CLIENT_ID=$(psqli -t -A -c "SELECT ad_client_id FROM ad_client WHERE name = '${CLIENT_NAME}';")
ROLE_ID=$(psqli -t -A -c "SELECT ad_role_id FROM ad_role WHERE name = '${CLIENT_NAME} Admin';")
ORG_ID=$(psqli -t -A -c "SELECT ad_org_id FROM ad_org WHERE value = '${ORG_VALUE}' AND ad_client_id = ${CLIENT_ID};")
BP_GROUP_ID=$(psqli -t -A -c "SELECT c_bp_group_id FROM c_bp_group WHERE ad_client_id = ${CLIENT_ID} AND isdefault = 'Y' LIMIT 1;")
```

## Foreign Key References

Three ways to reference foreign keys when creating/updating records:

```bash
# 1. Numeric ID (requires knowing the ID)
"C_BP_Group_ID": 1000000

# 2. Object with ID
"C_BP_Group_ID": {"id": 1000000}

# 3. Identifier lookup (recommended - uses the display name)
"C_BP_Group_ID": {"identifier": "Standard"}
"C_Country_ID": {"identifier": "United States"}
"C_Region_ID": {"identifier": "CA"}
```

The identifier lookup matches against the record's display identifier (typically Name or Value).

## Create Record

```bash
RESPONSE=$(curl -s -X POST "${API_URL}/models/${TABLE_NAME}" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{"Name": "...", "Value": "..."}')

# Extract ID from response
ID=$(echo "$RESPONSE" | grep -o '"id":[0-9]*' | head -1 | cut -d':' -f2)
```

## Nested Records

Create master + detail records in one call. Detail records can include inline creation of their own FKs.

```bash
# Create Business Partner with Contact and Location (all in one call)
curl -s -X POST "${API_URL}/models/C_BPartner" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{
        "Value": "CUST-001",
        "Name": "Test Customer",
        "IsCustomer": true,
        "IsVendor": false,
        "C_BP_Group_ID": {"identifier": "Standard"},
        "AD_User": [{
            "Name": "John Smith",
            "EMail": "john@example.com",
            "Phone": "408-555-1234"
        }],
        "C_BPartner_Location": [{
            "Name": "Main Office",
            "IsBillTo": true,
            "IsShipTo": true,
            "C_Location_ID": {
                "Address1": "123 Main St",
                "City": "San Jose",
                "Postal": "95110",
                "C_Country_ID": {"identifier": "United States"},
                "C_Region_ID": {"identifier": "CA"}
            }
        }]
    }'
```

## Run Process

### Process Slug Format

The process slug is generated by the database `Slugify()` function on `AD_Process.Value`:
- Converts to lowercase
- Converts spaces to dashes
- Underscores remain as underscores

```bash
# Find process slug from database using Slugify()
psqli -t -A -c "SELECT Slugify(value) FROM ad_process WHERE name = 'Open/Close All';"
# Returns: c_period_process

psqli -t -A -c "SELECT Slugify(value) FROM ad_process WHERE name = 'Process Order';"
# Returns: c_order-process (note: space in Value becomes dash)

# Or query via API
curl -s "${API_URL}/processes/c_period_process" -H "Authorization: Bearer $SESSION_TOKEN"
```

### Basic Process Execution

```bash
RESPONSE=$(curl -s -X POST "${API_URL}/processes/${PROCESS_SLUG}" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{"param1": "value1"}')

# Check for error
if echo "$RESPONSE" | grep -q '"isError":true'; then
    echo "Process failed"
fi
```

### Run Process on Specific Record (Button Process)

For processes that act on a specific record (like document actions):

```bash
# Open period (C_Period_Process on a C_Period record)
curl -s -X POST "${API_URL}/processes/c_period_process" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{
        "model-name": "c_period",
        "record-id": 1000000,
        "PeriodAction": "O"
    }'

# Complete order (C_Order Process -> slug: c_order-process)
curl -s -X POST "${API_URL}/processes/c_order-process" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{
        "model-name": "c_order",
        "record-id": 1000000,
        "DocAction": "CO"
    }'
```

**Special parameters for record-based processes:**
- `model-name`: Table name (lowercase, e.g., `c_order`, `c_period`)
- `record-id`: ID of the record to process

## Query Records

```bash
# Get by ID
curl -s "${API_URL}/models/C_BPartner/1000000" \
    -H "Authorization: Bearer $SESSION_TOKEN"

# Get by UUID
curl -s "${API_URL}/models/C_BPartner/2595c343-2e39-4a47-9a3d-0620fbaa566b" \
    -H "Authorization: Bearer $SESSION_TOKEN"

# Filter by field value
curl -s "${API_URL}/models/C_BPartner?\$filter=IsVendor eq true" \
    -H "Authorization: Bearer $SESSION_TOKEN"

# Filter by Name (lookup pattern)
curl -s "${API_URL}/models/C_BP_Group?\$filter=Name eq 'Standard'" \
    -H "Authorization: Bearer $SESSION_TOKEN"

# Filter with multiple conditions
curl -s "${API_URL}/models/C_BPartner?\$filter=IsCustomer eq true AND IsActive eq true" \
    -H "Authorization: Bearer $SESSION_TOKEN"

# Select specific fields and expand detail records
curl -s "${API_URL}/models/C_Order?\$select=DocumentNo,GrandTotal&\$expand=C_OrderLine" \
    -H "Authorization: Bearer $SESSION_TOKEN"

# Expand contacts and locations for a BP
curl -s "${API_URL}/models/C_BPartner/1000000?\$expand=AD_User,C_BPartner_Location" \
    -H "Authorization: Bearer $SESSION_TOKEN"
```

### Filter Operators

| Operator | Example |
|----------|---------|
| `eq` | `Name eq 'Standard'` |
| `neq` | `IsActive neq false` |
| `gt`, `ge`, `lt`, `le` | `GrandTotal gt 1000` |
| `and`, `or` | `IsVendor eq true AND IsActive eq true` |
| `contains()` | `contains(Name,'Stone')` |
| `startswith()` | `startswith(Value,'CUST')` |

## Error Checking

```bash
if [[ -z "$TOKEN" ]]; then
    echo "Authentication failed"
    echo "Response: $AUTH_RESPONSE"
    exit 1
fi

if echo "$RESPONSE" | grep -q '"isError":true'; then
    echo "API error"
    exit 1
fi
```

## Generating IDs and UUIDs in SQL Scripts

When inserting records directly via SQL (not through REST API), you need to generate primary key IDs and UUIDs.

**Prerequisite:** Native sequences must be enabled. In `idempiere-golive-deploy`, this is done by `20260107235500_enable_native_sequences.sh` which runs early in the deploy process.

### Native Sequences (Preferred)

After native sequences are enabled, use PostgreSQL `nextval()` with the table's sequence:

```sql
-- Sequence naming convention: lowercase_tablename_sq
SELECT nextval('ad_role_sq') INTO v_role_id;
SELECT nextval('c_bpartner_sq') INTO v_bp_id;
SELECT nextval('ad_importtemplate_sq') INTO v_template_id;
```

To find the sequence name for a table:
```bash
psqli -c "SELECT sequencename FROM pg_sequences WHERE sequencename = 'tablename_sq';"
```

### UUID Generation

Use `gen_random_uuid()` for UUID columns:

```sql
INSERT INTO ad_role (ad_role_id, ad_role_uu, name, ...)
VALUES (nextval('ad_role_sq'), gen_random_uuid()::varchar, 'My Role', ...);
```

### Legacy: nextidfunc (Avoid)

The function `nextidfunc(table_id, 'N')` operates on the AD_Sequence table. **Do not use this after native sequences are enabled** - the AD_Sequence table is not kept in sync with native sequences.

```sql
-- AVOID: Only use if native sequences are NOT enabled
SELECT nextidfunc(156, 'N') INTO v_role_id;  -- 156 = AD_Role table ID
```

### Example: Complete SQL Insert Pattern

```sql
DO $$
DECLARE
    v_record_id INTEGER;
BEGIN
    -- Get next ID from native sequence
    SELECT nextval('ad_role_sq') INTO v_record_id;

    -- Insert with generated ID and UUID
    INSERT INTO adempiere.ad_role (
        ad_role_id, ad_role_uu, ad_client_id, ad_org_id,
        name, isactive, created, createdby, updated, updatedby
    ) VALUES (
        v_record_id, gen_random_uuid()::varchar, 1000000, 0,
        'Department User', 'Y', now(), 100, now(), 100
    );

    RAISE NOTICE 'Created role with ID %', v_record_id;
END;
$$;
```

## Reference

Full API documentation: `idempiere-rest-docs/`

Working examples: `idempiere-golive-deploy/deploy/*.sh`

Tags: #tool #api #rest #idempiere
