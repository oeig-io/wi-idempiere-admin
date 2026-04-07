---
name: idempiere-groovy-deploy-pattern
description: Standard pattern for deploying Groovy scripts to iDempiere AD_Rule using .sh deploy scripts with temp files for reliable escaping
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-groovy-deploy-pattern-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Groovy Deploy Pattern

The standard pattern for deploying Groovy scripts (processes, callouts, model validators) into iDempiere's `ad_rule.script` column.

## The Problem

Groovy code must be stored in a PostgreSQL text column (`ad_rule.script`). Getting it there requires passing through multiple layers that each have escaping rules:

| Layer | Risk |
|-------|------|
| Bash strings | `$variable` expanded, single quotes break strings |
| envsubst | `deploy.sh` runs `.sql` files through envsubst — silently destroys `$groovyVar` and `${groovyVar}` |
| SQL string literals | Single quotes must be doubled (`'` → `''`) |
| PostgreSQL `$$` blocks | `$$` in Groovy conflicts with PL/pgSQL delimiters |

The `.sh` + temp file pattern eliminates all of these risks except SQL single-quote escaping, which is handled automatically by `sed`.

## The Pattern

All Groovy deploy scripts use `.sh` files (not `.sql`) with three steps:

### Step 1: Write Groovy to a temp file (quoted heredoc)

```bash
cat > /tmp/my_groovy.sql << 'GROOVYEOF'
import org.compiere.util.DB
import org.compiere.model.*

def ctx = A_Ctx
def name = "Hello"
def sql = "SELECT name FROM ad_client WHERE name = 'System'"
def result = DB.getSQLValueString(null, sql)
def blockStr = "test"
def interp = "Value is ${blockStr} here"

return result
GROOVYEOF
```

**Why this works:**
- The quoted `'GROOVYEOF'` prevents all bash expansion — `$blockStr`, `${blockStr}` are preserved literally
- Single quotes in Groovy (`'System'`) are written naturally — no manual escaping
- Comments, apostrophes, dollar signs — all safe

### Step 2: Escape single quotes for SQL and embed in SQL file

```bash
GROOVY_SCRIPT=$(cat /tmp/my_groovy.sql | sed "s/'/''/g")
rm /tmp/my_groovy.sql

cat > /tmp/my_deploy.sql << SQLEOF
DO \$\$
DECLARE
    v_rule_id numeric;
BEGIN
    v_rule_id := nextval('ad_rule_sq');

    INSERT INTO ad_rule (
        ad_rule_id, ad_client_id, ad_org_id, isactive, created, createdby,
        updated, updatedby, value, name, description,
        eventtype, ruletype, script, entitytype, ad_rule_uu, accesslevel
    ) VALUES (
        v_rule_id, 0, 0, 'Y', now(), 100, now(), 100,
        'groovy:MyScript',
        'My Script Name',
        'Description',
        'P', 'S',
        '$GROOVY_SCRIPT',
        'U', uuid_generate_v4()::varchar, '3'
    );
END \$\$;
SQLEOF
```

**Why this works:**
- `sed "s/'/''/g"` automatically doubles all single quotes for SQL — no manual escaping
- The unquoted `SQLEOF` allows `$GROOVY_SCRIPT` to expand into the SQL file
- `\$\$` escapes the PostgreSQL dollar quoting from bash expansion (only 2 occurrences)

### Step 3: Pipe to psqli

```bash
cat /tmp/my_deploy.sql | psqli
rm /tmp/my_deploy.sql
```

**Why this works:**
- `cat file | psqli` sends the SQL directly to PostgreSQL — no additional escaping layers
- envsubst never touches the content (only `.sql` files go through envsubst in `deploy.sh`)

## Round-Trip Verification

The pattern has been verified: writing Groovy through this pattern and reading it back from the database produces an identical file. Single quotes, `$variables`, `${interpolation}`, apostrophes — all preserved.

## Complete Example

A deploy script that creates a Groovy process with AD_Rule, AD_Process, and parameters:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo ">>> Creating Groovy process..."

# Step 1: Write clean Groovy
cat > /tmp/my_groovy.sql << 'GROOVYEOF'
import org.compiere.util.DB
import org.compiere.util.Env
import org.compiere.model.*

def ctx = A_Ctx
def trxName = A_TrxName
def lineId = A_Record_ID

def product = new MProduct(ctx, lineId, trxName)
if (product.get_ID() == 0) {
    return "@Error@ Product not found"
}

def catName = DB.getSQLValueString(trxName,
    "SELECT name FROM m_product_category WHERE m_product_category_id = ?",
    product.getM_Product_Category_ID())

return "@OK@ Category: " + catName
GROOVYEOF

# Step 2: Escape and embed in SQL
GROOVY_SCRIPT=$(cat /tmp/my_groovy.sql | sed "s/'/''/g")
rm /tmp/my_groovy.sql

cat > /tmp/my_process.sql << SQLEOF
DO \$\$
DECLARE
    v_rule_id numeric;
    v_process_id numeric;
BEGIN
    SELECT ad_process_id INTO v_process_id
    FROM ad_process WHERE value = 'MyProcess';

    IF v_process_id IS NOT NULL THEN
        RAISE NOTICE 'Process already exists, skipping';
        RETURN;
    END IF;

    v_rule_id := nextval('ad_rule_sq');
    INSERT INTO ad_rule (
        ad_rule_id, ad_client_id, ad_org_id, isactive, created, createdby,
        updated, updatedby, value, name, description,
        eventtype, ruletype, script, entitytype, ad_rule_uu, accesslevel
    ) VALUES (
        v_rule_id, 0, 0, 'Y', now(), 100, now(), 100,
        'groovy:MyProcess', 'My Process', 'Description',
        'P', 'S', '$GROOVY_SCRIPT', 'U', uuid_generate_v4()::varchar, '3'
    );

    v_process_id := nextval('ad_process_sq');
    INSERT INTO ad_process (
        ad_process_id, ad_client_id, ad_org_id, isactive, created, createdby,
        updated, updatedby, value, name, description,
        accesslevel, entitytype, isreport, isbetafunctionality,
        allowmultipleexecution, classname, ad_process_uu
    ) VALUES (
        v_process_id, 0, 0, 'Y', now(), 100, now(), 100,
        'MyProcess', 'My Process', 'Description',
        '3', 'U', 'N', 'N', 'N',
        '@script:groovy:MyProcess', uuid_generate_v4()::varchar
    );
END \$\$;
SQLEOF

# Step 3: Execute
cat /tmp/my_process.sql | psqli
rm /tmp/my_process.sql
```

## Non-Groovy SQL in .sh Scripts

For SQL that does not contain Groovy (e.g., AD_Element, AD_Column, AD_Field inserts), use `cat | psqli` with a quoted heredoc:

```bash
cat > /tmp/phase.sql << 'SQLEOF'
INSERT INTO ad_element (...)
SELECT ...
WHERE NOT EXISTS (...);
SQLEOF

cat /tmp/phase.sql | psqli
rm /tmp/phase.sql
```

For SQL that needs bash variable substitution (e.g., IDs from previous phases), use an unquoted heredoc:

```bash
cat > /tmp/phase.sql << SQLEOF
INSERT INTO ad_ref_list (... ad_reference_id ...)
SELECT ... $REF_ID ...
WHERE NOT EXISTS (...);
SQLEOF

cat /tmp/phase.sql | psqli
rm /tmp/phase.sql
```

## Applying to All Groovy Types

This pattern works for all three Groovy artifact types. The only difference is the `eventtype` in AD_Rule:

| Type | eventtype | Linked via |
|------|-----------|-----------|
| Process | `P` | AD_Process.classname → `@script:groovy:Name` |
| Callout | `C` | AD_Column.callout → `@script:groovy:Name` |
| Model Validator | `T` or `D` | AD_Table_ScriptValidator |

See the specific tool docs for AD_Process, AD_Column.callout, and AD_Table_ScriptValidator patterns:
- [idempiere-process-tool.md](idempiere-process-tool.md)
- [idempiere-callout-tool.md](idempiere-callout-tool.md)
- [idempiere-model-validator-tool.md](idempiere-model-validator-tool.md)

## Troubleshooting Legacy .sql Scripts

The sections below document issues found in older `.sql` deploy scripts that embed Groovy directly. These issues are avoided by the `.sh` pattern above, but are documented here for understanding legacy scripts.

### Unescaped Single Quotes in Embedded Scripts

**Symptom:** PostgreSQL error about "zero-length delimited identifier" or strange identifier truncation notices when running a `DO $$ ... END $$` block that contains a `v_script := '...'` string.

```
NOTICE:  identifier ""
    SELECT ...
"" will be truncated to "...
ERROR:  zero-length delimited identifier at or near """"
LINE 73:     return ""
```

**Cause:** An unescaped single quote (`'`) inside the `v_script` string literal prematurely terminates the string. PostgreSQL then tries to parse the remaining content as SQL, misinterpreting Groovy's `"""` triple quotes as PostgreSQL identifiers.

### envsubst Variable Conflicts

**Symptom:** Variables like `${something}` in your script are unexpectedly empty or replaced.

**Cause:** The deploy.sh runs `.sql` files through `envsubst` for variable substitution. If Groovy code contains `$variable` or `${variable}` syntax, envsubst replaces them with empty strings (or environment variable values).

**Known bug:** The slab breakdown script (`20260122000000_mr_line_slab_breakdown_process.sql`) has `$blockStr` in error messages that was silently replaced with empty string by envsubst.

### Dollar Quoting Conflicts

**Symptom:** Errors when using `$$` inside a `DO $$ ... END $$` block.

**Cause:** PostgreSQL's dollar quoting uses `$$` as delimiters. If Groovy code contains `$$`, it conflicts.

**Important:** Do NOT use tagged dollar quotes (e.g., `DO $script$`) in deploy scripts. The deploy.sh runs files through `envsubst` which replaces `$tag` with empty string.

### Groovy String Interpolation and Dollar Signs

**Symptom:** PostgreSQL parse errors when using Groovy string interpolation (`$variable`) inside SQL-embedded scripts.

**Cause:** PostgreSQL interprets `$variable` as dollar quoting, conflicting with Groovy's string interpolation syntax.

### Groovy Comments with Special Characters

**Symptom:** "zero-length delimited identifier" errors or truncation notices.

**Cause:** Groovy comments containing apostrophes, double quotes, dollar signs, or backticks break the SQL string literal when embedded in `v_script := '...'` or inline in `DO $$ ... END $$` blocks.

Tags: #tool #idempiere #groovy #deploy-pattern
