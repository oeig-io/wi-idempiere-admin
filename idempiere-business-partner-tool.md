---
name: idempiere-business-partner
description: Creating and managing business partners, contacts, and login users in iDempiere with proper org assignments and role management
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-business-partner-tool.md
  category: data
  scope: idempiere
---

# iDempiere Business Partner Tool

The purpose of this tool is to define how ACME creates and manages business partners (customers, vendors, employees) and their associated contacts/login users.

This is important because proper BP setup ensures correct accounting, access control, and financial engagement tracking across all organizations.

This document references the iDempiere REST API. See [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) for authentication patterns and API usage.

## TOC

- [Core Principles](#core-principles)
- [Business Partner Types](#business-partner-types)
- [Creation Rules](#creation-rules)
  - [Key Fields](#key-fields)
- [BP with Contact](#bp-with-contact)
  - [SQL Creation](#sql-creation)
  - [REST API Creation](#rest-api-creation)
- [Contact with Login Access](#contact-with-login-access)
  - [During Deploy/ (SQL Method)](#during-deploy-sql-method)
  - [Post-Deploy (API Method)](#post-deploy-api-method)
- [Standalone Login Users](#standalone-login-users)
  - [Creation](#creation)
- [SQL vs API Decision](#sql-vs-api-decision)
- [BP Accounting Records (After-Save)](#bp-accounting-records-after-save)
  - [Detecting the gap](#detecting-the-gap)
  - [Fix: run Copy Accounts](#fix-run-copy-accounts)
- [Examples](#examples)
  - [Example 1: Simple Vendor with Contact](#example-1-simple-vendor-with-contact)
  - [Example 2: Employee with iDempiere Access](#example-2-employee-with-idempiere-access)
  - [Example 3: Standalone Admin User](#example-3-standalone-admin-user)
- [Password Management](#password-management)
  - [During Deploy (Before Hashing)](#during-deploy-before-hashing)
  - [Post-Deploy (After Hashing Enabled)](#post-deploy-after-hashing-enabled)
- [Verification](#verification)

## Core Principles

All business partner records follow these immutable rules:

- **BP Definition**: Anyone we engage with financially (customers, vendors, employees)
- **Organization Independence**: All BPs and contacts use `ad_org_id=0` (accessible across all orgs)
- **BP Groups**: Use "Standard" group unless business rules specify otherwise
- **Single AD_User**: One record serves as both contact and login (if needed)
- **Accounting Records**: The `c_bpartner` row must always be created through the model layer (REST API `.sh`) so the `MBPartner` after-save event generates accounting records. Direct SQL `INSERT` into `c_bpartner` skips this. See [BP Accounting Records (After-Save)](#bp-accounting-records-after-save).

## Business Partner Types

A BP can be any combination of:

| Type | Flag | Purpose |
|------|------|---------|
| Customer | `iscustomer='Y'` | We sell to them |
| Vendor | `isvendor='Y'` | We buy from them |
| Employee | `isemployee='Y'` | Works for us |

**Multi-type BPs**: Any combination is valid (e.g., employee who is also a vendor for expense reimbursement).

## Creation Rules

| Record | ad_org_id | c_bpartner_id | Can Login |
|--------|-----------|---------------|-----------|
| Business Partner | 0 | N/A | N/A |
| BP Contact | 0 | Set to BP ID | No (unless role added) |
| Login User | 0 | NULL or BP ID | Yes (requires role + password) |

### Key Fields

- `value` - Unique identifier (e.g., "STONE-SUPPLIER", "brian.sweeney")
- `name` - Display name (e.g., "Stone Supplier Inc", "Brian Sweeney")
- `ad_org_id` - Always 0 for BP records
- `isactive` - 'Y' unless stated otherwise

## BP with Contact

Almost all BPs have at least one contact. The contact is an `ad_user` record linked to the BP.

**Contact-Only (No Login):**
- `ad_user.c_bpartner_id` = BP ID
- No records in `ad_user_roles`
- No password needed

**Example:** Factory worker who appears on timesheets but never logs into iDempiere.

### SQL Creation

```sql
-- 1. Create BP
INSERT INTO c_bpartner (
    c_bpartner_id, ad_client_id, ad_org_id, isactive,
    value, name, c_bp_group_id,
    isemployee, iscustomer, isvendor,
    c_bpartner_uu, created, createdby, updated, updatedby
) VALUES (
    nextval('c_bpartner_sq'), ${CLIENT_ID}, 0, 'Y',
    '${BP_VALUE}', '${BP_NAME}',
    (SELECT c_bp_group_id FROM c_bp_group WHERE name = 'Standard' AND ad_client_id = ${CLIENT_ID}),
    '${IS_EMPLOYEE}', '${IS_CUSTOMER}', '${IS_VENDOR}',
    uuid_generate_v4(), now(), 100, now(), 100
);

-- 2. Create Contact (no login)
INSERT INTO ad_user (
    ad_user_id, ad_client_id, ad_org_id, isactive,
    c_bpartner_id, value, name, email,
    ad_user_uu, created, createdby, updated, updatedby
) VALUES (
    nextval('ad_user_sq'), ${CLIENT_ID}, 0, 'Y',
    ${BP_ID}, '${CONTACT_VALUE}', '${CONTACT_NAME}', '${EMAIL}',
    uuid_generate_v4(), now(), 100, now(), 100
);
```

### REST API Creation

```json
POST /api/v1/models/c_bpartner
{
  "AD_Org_ID": {"id": 0},
  "Value": "STONE-SUPPLIER",
  "Name": "Stone Supplier Inc",
  "IsVendor": true,
  "C_BP_Group_ID": {"identifier": "Standard"},
  "AD_User": [{
    "AD_Org_ID": {"id": 0},
    "Name": "Mike Stone",
    "EMail": "mike@stonesupplier.example.com"
  }]
}
```

## Contact with Login Access

A BP contact who needs to log into iDempiere requires:
1. BP record (for financial tracking)
2. Contact record (`ad_user` with `c_bpartner_id` set)
3. **Plus**: Role assignment + password

**Example:** Manager who is an employee (BP) and needs iDempiere access.

### During Deploy/ (SQL Method)

Use SQL with plain text password. The deploy/20261231235900_hash_passwords.sh script hashes all passwords as a final step.

```sql
-- 1. Create BP (employee)
INSERT INTO c_bpartner (...) VALUES (...);

-- 2. Create Contact
INSERT INTO ad_user (...) VALUES (...);

-- 3. Assign Role (enables login)
INSERT INTO ad_user_roles (
    ad_user_roles_uu, ad_client_id, ad_org_id,
    ad_user_id, ad_role_id, isactive,
    created, createdby, updated, updatedby
) VALUES (
    uuid_generate_v4(), ${CLIENT_ID}, 0,
    ${USER_ID}, ${ROLE_ID}, 'Y',
    now(), 100, now(), 100
);

-- Note: Password set in ad_user.password will be auto-hashed by deploy process
```

### Post-Deploy (API Method)

Use REST API. The API handles password hashing automatically.

```json
POST /api/v1/models/ad_user
{
  "AD_Org_ID": {"id": 0},
  "C_BPartner_ID": {"id": 1000208},
  "Value": "brian.sweeney",
  "Name": "Brian Sweeney",
  "EMail": "user@example.com",
  "Password": "[SET_PASSWORD]"
}
```

Then assign role:

```json
POST /api/v1/models/ad_user_roles
{
  "AD_Org_ID": {"id": 0},
  "AD_User_ID": {"id": 1000030},
  "AD_Role_ID": {"id": 1000001}
}
```

## Standalone Login Users

**No BP association** - users who need iDempiere access but aren't tracked as BPs.

**Example:** System administrators, external consultants.

Characteristics:
- `ad_user.c_bpartner_id` = NULL
- `ad_user.ad_org_id` = 0
- Has at least one role
- Has password

### Creation

**During deploy/ (SQL):**

```sql
INSERT INTO ad_user (
    ad_user_id, ad_client_id, ad_org_id, isactive,
    value, name, email, password,
    ad_user_uu, created, createdby, updated, updatedby
) VALUES (
    nextval('ad_user_sq'), ${CLIENT_ID}, 0, 'Y',
    '${USERNAME}', '${FULL_NAME}', '${EMAIL}', '${PASSWORD}',
    uuid_generate_v4(), now(), 100, now(), 100
);

INSERT INTO ad_user_roles (...) VALUES (...);
```

**Post-deploy (API):**

```json
POST /api/v1/models/ad_user
{
  "AD_Org_ID": {"id": 0},
  "Value": "admin.user",
  "Name": "Admin User",
  "EMail": "admin@example.com",
  "Password": "[SET_PASSWORD]"
}
```

## SQL vs API Decision

> ⚠️ **Warning** - This choice applies to the `ad_user` / `ad_user_roles` rows. The `c_bpartner` row itself should be created through the **model layer** (REST API `.sh`) whenever possible, because the `MBPartner` after-save event creates the BP's accounting records. If you must create `c_bpartner` via SQL (e.g., large batch imports), you **must** run the Copy Accounts process afterward — see [BP Accounting Records (After-Save)](#bp-accounting-records-after-save).

| Scenario | `c_bpartner` Method | `ad_user` Method | Reason |
|----------|--------|--------|--------|
| Deploy/ phase scripts | REST API `.sh` (preferred) | SQL | Model layer creates accounting; passwords auto-hashed by deploy/20261231235900_hash_passwords.sh |
| Deploy/ batch imports (many BPs) | SQL + Copy Accounts | SQL | SQL is faster; Copy Accounts backfills the accounting records the after-save event would have created |
| Post-deployment | REST API | REST API | API handles password hashing and after-save events |
| Interactive creation | REST API | REST API | Real-time validation, after-save events, immediate feedback |

## BP Accounting Records (After-Save)

When a `c_bpartner` is created through the **model layer** (REST API, UI, or import process), `MBPartner`'s after-save event automatically creates its per-BP accounting records (`c_bp_customer_acct`, `c_bp_vendor_acct`) from the BP Group's accounting defaults.

**Direct SQL `INSERT` into `c_bpartner` bypasses this event**, leaving the BP with **no accounting records**. Such a BP cannot be used in posted transactions correctly until its accounting is generated.

### Detecting the gap

```sql
-- BPs missing customer accounting records
SELECT bp.value, bp.name
FROM c_bpartner bp
WHERE bp.ad_client_id = ${CLIENT_ID}
  AND NOT EXISTS (
    SELECT 1 FROM c_bp_customer_acct c WHERE c.c_bpartner_id = bp.c_bpartner_id
  );
```

### Fix: run Copy Accounts

The core **Copy Accounts** process (`c_bp_group_acct_copy`) copies the BP Group's account template onto every BP in the group, inserting any missing per-BP accounting records. It is idempotent — it only inserts what is missing. Run it via REST API in a `.sh` deploy step after any SQL-based BP creation:

```bash
# Authenticate as <CLIENT>-admin, select client context, then per BP group:
POST /api/v1/processes/c_bp_group_acct_copy
{
  "C_BP_Group_ID": <bp_group_id>,
  "C_AcctSchema_ID": <acct_schema_id>
}
```

Result summary looks like `Created=20, Updated=22` (records inserted/refreshed). Verify afterward that every BP has `c_bp_customer_acct` and `c_bp_vendor_acct` rows.

> 🔗 **Reference** - Working example: `idempiere-golive-deploy/deploy/20260601195707_copy_bp_group_accounts.sh` (and the ANS repo's `20260522000400_import_customers.sh`, Step 4).

## Examples

### Example 1: Simple Vendor with Contact

Vendor who receives POs but doesn't log in:

```sql
-- BP
INSERT INTO c_bpartner (...) VALUES (1000208, [CLIENT_ID], 0, 'Y', 'STONE-SUPPLIER', 'Stone Supplier Inc', [BP_GROUP_ID], 'N', 'N', 'Y', ...);

-- Contact (no login)
INSERT INTO ad_user (...) VALUES (1000002, [CLIENT_ID], 0, 'Y', 1000208, 'mike.stone', 'Mike Stone', 'mike@stonesupplier.example.com', ...);
```

### Example 2: Employee with iDempiere Access

Employee who needs to log in as ACME Admin:

```sql
-- BP
INSERT INTO c_bpartner (...) VALUES (1000210, [CLIENT_ID], 0, 'Y', 'brian.sweeney', 'Brian Sweeney', [BP_GROUP_ID], 'Y', 'N', 'N', ...);

-- Contact with login
INSERT INTO ad_user (...) VALUES (1000030, [CLIENT_ID], 0, 'Y', 1000210, 'brian.sweeney', 'Brian Sweeney', 'brian.sweeney@example.com', 'brian.sweeney', ...);

-- Role assignment (enables login)
INSERT INTO ad_user_roles (...) VALUES (1000030, [CLIENT_ID], 0, 1000030, 1000001, 'Y', ...); -- ACME Admin role
```

### Example 3: Standalone Admin User

System administrator not tracked as employee:

```sql
-- Login user only (no BP)
INSERT INTO ad_user (...) VALUES (1000031, [CLIENT_ID], 0, 'Y', NULL, 'admin-user', 'ACME-admin', 'admin@example.com', 'ACME-admin', ...);

-- Role assignment
INSERT INTO ad_user_roles (...) VALUES (1000031, [CLIENT_ID], 0, 1000031, 1000001, 'Y', ...); -- ACME Admin role
```

## Password Management

### During Deploy (Before Hashing)

Set passwords as plain text in SQL. The `deploy/20261231235900_hash_passwords.sh` script converts all plain text passwords to hashed values as a final step.

### Post-Deploy (After Hashing Enabled)

Once `USER_PASSWORD_HASH` sysconfig is `Y`, passwords cannot be set via direct SQL. Use `PUT` on the `ad_user` model via REST API with `{"Password":"newpassword"}` — iDempiere's MUser model layer hashes automatically. See "Update Record" in the REST API skill.

> 📝 **Note** - The old `Reset Password` process (`AD_User_Password`, ID 288) is inactive. The REST API `PUT` on `ad_user` is the replacement.


## Verification

Check user can log in:

```sql
-- Should return at least one role
SELECT r.name 
FROM ad_user_roles ur 
JOIN ad_role r ON ur.ad_role_id = r.ad_role_id 
WHERE ur.ad_user_id = ${USER_ID};

-- Should have password (will be hashed after deploy)
SELECT name, password IS NOT NULL as has_password 
FROM ad_user 
WHERE ad_user_id = ${USER_ID};
```

Tags: #tool #bpartner #idempiere #data-management #user-management
