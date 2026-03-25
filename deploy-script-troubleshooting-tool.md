---
name: deploy-script-troubleshooting
description: Common issues and solutions when writing SQL deploy scripts for iDempiere including unescaped quotes, delimiter problems, and identifier issues
compatibility: opencode
metadata:
  type: tool
  original_file: deploy-script-troubleshooting-tool.md
  category: debugging
  scope: idempiere
---

# Deploy Script Troubleshooting

Common issues when writing SQL deploy scripts for iDempiere.

## Unescaped Single Quotes in Embedded Scripts

**Symptom:** PostgreSQL error about "zero-length delimited identifier" or strange identifier truncation notices when running a `DO $$ ... END $$` block that contains a `v_script := '...'` string.

```
NOTICE:  identifier ""
    SELECT ...
"" will be truncated to "...
ERROR:  zero-length delimited identifier at or near """"
LINE 73:     return ""
```

**Cause:** An unescaped single quote (`'`) inside the `v_script` string literal prematurely terminates the string. PostgreSQL then tries to parse the remaining content as SQL, misinterpreting Groovy's `"""` triple quotes as PostgreSQL identifiers.

**Example - WRONG:**
```sql
v_script := '
// Use PO's transaction name   <-- apostrophe breaks the string!
def trxName = ol.get_TrxName()
';
```

**Example - CORRECT:**
```sql
v_script := '
// Use PO transaction name     <-- apostrophe removed
def trxName = ol.get_TrxName()
';
```

**Alternative fix - escape the quote:**
```sql
v_script := '
// Use PO''s transaction name  <-- doubled single quote
def trxName = ol.get_TrxName()
';
```

**How to check for this issue:**
```bash
# Find unescaped single quotes inside v_script blocks
for f in deploy/*.sql; do
  result=$(sed -n '/v_script := /,/^'"'"';$/p' "$f" 2>/dev/null | grep -v "^'" | grep -v "^    v_script" | grep "'")
  if [[ -n "$result" ]]; then
    echo "=== $(basename "$f") ==="
    echo "$result"
  fi
done
```

## CRLF Line Endings

**Symptom:** Scripts fail with `$'\r': command not found` or similar errors.

**Cause:** Files created on Windows or by some tools have CRLF (`\r\n`) line endings instead of Unix LF (`\n`).

**Fix:**
```bash
sed -i 's/\r$//' deploy/script.sql
```

**Prevention:** The repository's `.gitattributes` enforces LF on checkout, but newly created files may still have CRLF until committed.

## envsubst Variable Conflicts

**Symptom:** Variables like `${something}` in your script are unexpectedly empty or replaced.

**Cause:** The deploy.sh runs SQL files through `envsubst` for variable substitution. If your script contains shell-like variable syntax that shouldn't be substituted, it will be replaced.

**Example - WRONG:**
```sql
-- In Groovy script
def result = "${someGroovyVar}"  <-- envsubst will try to replace this
```

**Fix:** Use Groovy string concatenation instead of GString interpolation when the variable doesn't exist in deploy.properties:
```sql
def result = "" + someGroovyVar
```

## Dollar Quoting Conflicts

**Symptom:** Errors when using `$$` inside a `DO $$ ... END $$` block.

**Cause:** PostgreSQL's dollar quoting uses `$$` as delimiters. If your embedded script contains `$$`, it conflicts.

**Fix:** Use a tagged dollar quote:
```sql
DO $script$
DECLARE
    v_script TEXT;
BEGIN
    v_script := '
    // Script content that might contain $$
    ';
END $script$;
```

## Groovy String Interpolation and Dollar Signs

**Symptom:** PostgreSQL parse errors when using Groovy string interpolation (`$variable`) inside SQL-embedded scripts.

```
ERROR:  syntax error at or near "$"
LINE 42:            "Cost: $" + costUsd + nl +
```

**Cause:** PostgreSQL interprets `$variable` as dollar quoting, conflicting with Groovy's string interpolation syntax.

**Fix - Avoid `$` entirely in embedded Groovy:**
```groovy
// WRONG - triggers PostgreSQL dollar quoting
"Cost: $" + costUsd

// CORRECT - use string concatenation without $
"Cost: USD " + costUsd
"Cost: " + costUsd + " USD"
```

**Fix - Use single-quoted strings (no interpolation):**
```groovy
// Single quotes = literal string, no $ escaping issues
'Cost: USD ' + costUsd
```

**Note:** This applies to Groovy scripts embedded in SQL via `v_script := '...'` or inline in `DO $$` blocks.

## Groovy Comments with Special Characters

**Symptom:** Similar to unescaped quotes - "zero-length delimited identifier" errors or strange truncation notices when deploying Groovy scripts.

**Cause:** Groovy comments (lines starting with `//` or `--`) containing special characters break the SQL string literal when embedded in the `v_script := '...'` format or when using inline Groovy in `DO $$ ... END $$` blocks.

**Problematic characters in comments:**
- Apostrophes (`'`, `''`)
- Double quotes (`"`)
- Dollar signs (`$`)
- Backticks (`` ` ``)
- Any character that has special meaning in SQL or PostgreSQL dollar quoting

**Example - WRONG:**
```sql
-- In Groovy script (inline in DO $$ block)
def sql = """
    -- ASI owner lookup: storageonhand -> locator -> warehouse -> org -> org's linked BP
    LEFT JOIN ...
"""
```

**Quick Fix - Delete ALL Comments:**
When encountering deployment issues with Groovy scripts, the **first troubleshooting step** should be to remove **all** comments from the embedded Groovy code. Comments are not required for the script to function.

```bash
# Remove // comments from embedded Groovy in a deploy script
sed -i '/^\s*\/\//d' deploy/your_script.sql

# Also remove -- comments if present
sed -i '/^\s*--/d' deploy/your_script.sql
```

**Prevention:** When writing Groovy scripts for iDempiere deploy scripts:
1. **Avoid comments entirely** - simplest solution
2. If comments are needed, ensure all special characters are properly escaped (complex and error-prone)
