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
