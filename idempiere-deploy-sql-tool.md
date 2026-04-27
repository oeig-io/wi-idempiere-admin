---
name: idempiere-deploy-sql
description: Conventions and patterns for writing SQL deploy scripts that are portable across iDempiere environments via deploy.sh variable substitution and dynamic lookups
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-deploy-sql-tool.md
  category: deployment
  scope: idempiere
---

# iDempiere Deploy SQL Tool

## TOC

- [Overview](#overview)
- [When to Use .sql vs .sh](#when-to-use-sql-vs-sh)
- [How deploy.sh Executes SQL](#how-deploysh-executes-sql)
- [Available Variables](#available-variables)
- [Client ID Lookup](#client-id-lookup)
- [Role Lookup](#role-lookup)
- [Primary Key Generation](#primary-key-generation)
- [UUID Generation](#uuid-generation)
- [Organization ID](#organization-id)
- [Idempotency](#idempotency)
- [System vs Tenant Records](#system-vs-tenant-records)
- [Nullable Columns](#nullable-columns)
- [File Naming](#file-naming)
- [Script Structure Template](#script-structure-template)
- [Common Mistakes](#common-mistakes)

## Overview

The purpose of this document is to define the conventions for writing `.sql` deploy scripts in `idempiere-golive-deploy/deploy/`.

This is important because deploy scripts must be portable across environments. Hardcoded IDs (client, role, org) break when a script runs against a different iDempiere instance where those IDs differ. The conventions here ensure scripts work reliably in any environment created by `deploy.sh`.

## When to Use .sql vs .sh

Deploy scripts are either `.sql` or `.sh`. The choice depends on what the script needs to do:

| Use `.sql` when | Use `.sh` when |
|---|---|
| Only database INSERTs/UPDATEs | REST API calls needed (table create, column sync, process execution) |
| No `$variable` or `${variable}` in content | Groovy code that contains `$` or `${...}` (envsubst destroys these) |
| Simple tenant-level configuration | Multi-phase scripts that mix SQL and API calls |
| No bash logic needed | Conditional logic, loops, or variable capture between steps |

> 🔗 **Reference** - For scripts containing Groovy code, always use `.sh`. See [idempiere-groovy-deploy-pattern-tool.md](idempiere-groovy-deploy-pattern-tool.md) for the standard pattern that avoids envsubst and escaping issues.

> 🔗 **Reference** - For scripts that create Application Dictionary objects (tables, columns, windows), use `.sh` with REST API calls. See [idempiere-table-create-tool.md](idempiere-table-create-tool.md) and [idempiere-column-create-tool.md](idempiere-column-create-tool.md).

## How deploy.sh Executes SQL

The `deploy.sh` pipeline for `.sql` files:

```
envsubst < local_file.sql | psqli -v ON_ERROR_STOP=1
```

1. `deploy.properties` is sourced as environment variables
2. `envsubst` substitutes `${VARIABLE}` references in the SQL file
3. The result is piped to `psqli` inside the container

> ⚠️ **Warning** - `envsubst` replaces ALL `${...}` patterns. If your SQL contains literal `${something}` (e.g., Groovy code), it will be silently replaced with an empty string. Use `.sh` scripts for Groovy content (see idempiere-groovy-deploy-pattern-tool.md).

## Available Variables

Variables come from `deploy.properties` and are substituted by `envsubst`:

| Variable | Example Value | Purpose |
|----------|---------------|---------|
| `${CLIENT_NAME}` | `ACME` | Tenant name (AD_Client.Name) |
| `${ORG_VALUE}` | `hwd` | Primary org value |
| `${ORG_NAME}` | `Hayward` | Primary org name |
| `${ADMIN_USER}` | `ACME-admin` | Admin username |
| `${NORMAL_USER}` | `ACME-user` | Normal username |
| `${CONTAINER_NAME}` | `id-47` | Container name (injected by deploy.sh) |

> 📝 **Note** - `${CLIENT_NAME}` is the most frequently used variable. It drives lookups for AD_Client_ID, role IDs, and user IDs.

> 🔗 **Reference** - `.sh` scripts receive these same variables as sourced environment variables from deploy.properties. See idempiere-golive-deploy CLAUDE.md for `.sh` execution details.

## Client ID Lookup

Never hardcode `ad_client_id`. Always look it up from `${CLIENT_NAME}`:

```sql
DO $$
DECLARE
    v_client_id numeric;
BEGIN
    SELECT ad_client_id INTO v_client_id
    FROM ad_client
    WHERE name = '${CLIENT_NAME}';

    IF v_client_id IS NULL THEN
        RAISE EXCEPTION '${CLIENT_NAME} client not found';
    END IF;

    -- Use v_client_id in all subsequent statements
    INSERT INTO some_table (ad_client_id, ...) VALUES (v_client_id, ...);
END $$;
```

For simple UPDATE/SELECT statements outside a DO block, use an inline subquery:

```sql
UPDATE c_doctype
SET name = 'New Name'
WHERE ad_client_id = (SELECT ad_client_id FROM ad_client WHERE name = '${CLIENT_NAME}')
  AND docbasetype = 'SOO';
```

## Role Lookup

Never hardcode `ad_role_id`. Role names follow the convention `${CLIENT_NAME} Admin` and `${CLIENT_NAME} User`:

```sql
SELECT ad_role_id INTO v_admin_role_id
FROM ad_role
WHERE name = '${CLIENT_NAME} Admin'
  AND ad_client_id = v_client_id;

SELECT ad_role_id INTO v_user_role_id
FROM ad_role
WHERE name = '${CLIENT_NAME} User'
  AND ad_client_id = v_client_id;
```

## Primary Key Generation

Never hardcode primary key IDs. Use the table's native sequence:

```sql
-- Pattern: tablename_sq (lowercase, no schema prefix)
nextval('pa_documentstatus_sq')
nextval('c_uom_sq')
nextval('ad_rule_sq')
nextval('ad_process_sq')
```

Usage in INSERT:

```sql
DECLARE
    v_record_id numeric;
BEGIN
    v_record_id := nextval('pa_documentstatus_sq');

    INSERT INTO pa_documentstatus (
        pa_documentstatus_id, ...
    ) VALUES (
        v_record_id, ...
    );
END $$;
```

> 📝 **Note** - The first deploy script (`20260107235400_system_api_role.sql`) runs before native sequences are enabled, so it reads from `ad_sequence` directly. All scripts after `20260107235500_enable_native_sequences.sh` use `nextval()`.

## UUID Generation

Never hardcode UUID strings. Use `gen_random_uuid()`:

```sql
INSERT INTO some_table (
    some_table_uu, ...
) VALUES (
    gen_random_uuid(), ...
);
```

> 📝 **Note** - The first deploy script uses `adempiere.uuid_generate_v4()` with schema prefix because it runs in a pre-native-sequence context. All later scripts use `gen_random_uuid()` without schema prefix.

## Organization ID

`ad_org_id = 0` means "all organizations" (`*`). This is correct for most configuration records. Use 0 directly — it is a constant, not an environment-specific value:

```sql
INSERT INTO some_table (ad_client_id, ad_org_id, ...)
VALUES (v_client_id, 0, ...);
```

When a record must be org-specific, look up the org by value or name:

```sql
SELECT ad_org_id INTO v_org_id
FROM ad_org
WHERE value = '${ORG_VALUE}'
  AND ad_client_id = v_client_id;
```

## Idempotency

Scripts should be safe to run multiple times. Use existence checks, not `ON CONFLICT` with hardcoded IDs:

```sql
-- Preferred: guard with existence check
SELECT some_id INTO v_existing_id
FROM some_table
WHERE name = 'My Record'
  AND ad_client_id = v_client_id;

IF v_existing_id IS NULL THEN
    v_new_id := nextval('some_table_sq');
    INSERT INTO some_table (...) VALUES (v_new_id, ...);
    RAISE NOTICE 'Created record: %', v_new_id;
ELSE
    RAISE NOTICE 'Record already exists (ID: %), skipping', v_existing_id;
END IF;
```

For simple cases, `INSERT ... WHERE NOT EXISTS` also works:

```sql
INSERT INTO some_table (some_table_id, ad_client_id, ...)
SELECT nextval('some_table_sq'), v_client_id, ...
WHERE NOT EXISTS (
    SELECT 1 FROM some_table
    WHERE name = 'My Record' AND ad_client_id = v_client_id
);
```

> ⚠️ **Warning** - Avoid `ON CONFLICT (primary_key_id) DO UPDATE` with hardcoded IDs. The same primary key value may already be used by a different record in another environment.

## System vs Tenant Records

The `ad_client_id` determines the record's ownership scope:

| AD_Client_ID | Scope | Use When |
|---|---|---|
| `0` | System | Record is part of iDempiere core or applies to all tenants |
| `v_client_id` | Tenant | Record is specific to the tenant being configured |

Deploy scripts create **tenant-level** records. The tenant is being built by deploy.sh, so records belong to it:

```sql
-- Correct: tenant-level record
INSERT INTO pa_documentstatus (ad_client_id, ...) VALUES (v_client_id, ...);

-- Correct: tenant-level access record
INSERT INTO pa_documentstatusaccess (ad_client_id, ...) VALUES (v_client_id, ...);
```

System-level records (AD_Client_ID=0) are only appropriate for:
- Application Dictionary changes (AD_Column, AD_Field, AD_Process, etc.)
- System configuration (AD_SysConfig at client 0)
- Records that must be visible to all tenants

> 🔗 **Reference** - See Standard INSERT Pattern in [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md) for the AD-level (System) INSERT conventions. The patterns here extend that for tenant-level deploy scripts.

## Nullable Columns

When a column should be NULL, use NULL explicitly. Do not substitute 0 for NULL when the semantics differ:

```sql
-- Correct: NULL means "not set"
INSERT INTO pa_documentstatusaccess (..., ad_user_id, ...)
VALUES (..., NULL, ...);

-- Wrong: 0 means "System user" which is a different meaning
INSERT INTO pa_documentstatusaccess (..., ad_user_id, ...)
VALUES (..., 0, ...);
```

## File Naming

Deploy scripts use datestamp prefixes for execution ordering:

```
YYYYMMDDHHMMSS_description.sql
```

Generate a timestamp with:

```bash
date +%Y%m%d%H%M%S
```

## Script Structure Template

A complete tenant-level deploy script:

```sql
-- Brief description of what this script does
--
-- Details about the business purpose and what records are created.

DO $$
DECLARE
    v_client_id numeric;
    v_admin_role_id numeric;
    v_user_role_id numeric;
    v_record_id numeric;
BEGIN
    -- Resolve tenant
    SELECT ad_client_id INTO v_client_id
    FROM ad_client WHERE name = '${CLIENT_NAME}';

    IF v_client_id IS NULL THEN
        RAISE EXCEPTION '${CLIENT_NAME} client not found';
    END IF;

    -- Resolve roles (when needed)
    SELECT ad_role_id INTO v_admin_role_id
    FROM ad_role
    WHERE name = '${CLIENT_NAME} Admin' AND ad_client_id = v_client_id;

    SELECT ad_role_id INTO v_user_role_id
    FROM ad_role
    WHERE name = '${CLIENT_NAME} User' AND ad_client_id = v_client_id;

    -- Check idempotency
    SELECT some_id INTO v_record_id
    FROM some_table
    WHERE name = 'My Record' AND ad_client_id = v_client_id;

    IF v_record_id IS NOT NULL THEN
        RAISE NOTICE 'Record already exists (ID: %), skipping', v_record_id;
        RETURN;
    END IF;

    -- Create record
    v_record_id := nextval('some_table_sq');
    INSERT INTO some_table (
        some_table_id, ad_client_id, ad_org_id, isactive,
        created, createdby, updated, updatedby,
        name, ...,
        some_table_uu
    ) VALUES (
        v_record_id, v_client_id, 0, 'Y',
        now(), 100, now(), 100,
        'My Record', ...,
        gen_random_uuid()
    );

    RAISE NOTICE 'Created record: %', v_record_id;
END $$;
```

## Common Mistakes

| Mistake | Correct |
|---|---|
| `ad_client_id = 11` | `ad_client_id = v_client_id` (looked up from `${CLIENT_NAME}`) |
| `ad_role_id = 1000001` | Looked up from `'${CLIENT_NAME} Admin'` |
| `pa_documentstatus_id = 200016` | `nextval('pa_documentstatus_sq')` |
| `pa_documentstatus_uu = '6a7c...'` | `gen_random_uuid()` |
| `ON CONFLICT (hardcoded_id)` | `IF v_existing IS NULL THEN INSERT` |
| `ad_user_id = 0` (when meaning NULL) | `ad_user_id = NULL` |
| `WHERE name = 'ANS'` | `WHERE name = '${CLIENT_NAME}'` |

Tags: #tool #idempiere #deploy #sql #conventions
