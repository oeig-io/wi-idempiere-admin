---
name: idempiere-code-validator
description: Audit Groovy and SQL code for resource leaks, connection management issues, and coding conventions before iDempiere deployment
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-code-validator-tool.md
  category: debugging
  scope: idempiere
---

# iDempiere Code Validator Tool

The purpose of this tool is to audit Groovy and SQL code for common issues before deployment.

This is important because iDempiere Groovy scripts run in the application server context where resource leaks (unclosed connections, statements, result sets) can degrade performance or cause failures.

## TOC

- [Quick Start](#quick-start)
- [Validation Targets](#validation-targets)
- [Checks](#checks)
  - [Groovy Resource Management](#groovy-resource-management)
- [Adding New Checks](#adding-new-checks)

## Quick Start

**Validate deploy scripts (local files):**
```
Review all Groovy code in idempiere-golive-deploy/deploy/ for resource management issues
```

**Validate deployed code (container):**
```
Review Groovy scripts in container id-44 by querying AD_Rule table for resource management issues
```

## Validation Targets

| Target | Location | Method |
|--------|----------|--------|
| Deploy scripts | `idempiere-golive-deploy/deploy/*.sh` and `*.sql` | Read embedded Groovy in deploy files |
| Container rules | Container `id-xx` | Query `AD_Rule` table via `psqli` |

**Query deployed Groovy scripts:**
```bash
incus exec id-44 -- sudo -u idempiere psqli -c "SELECT value, script FROM ad_rule WHERE ruletype = 'S' AND script IS NOT NULL"
```

## Checks

### Groovy Resource Management

**Issue:** JDBC resources (PreparedStatement, ResultSet) not closed, causing connection pool exhaustion.

**What to look for:**

| Pattern | Status | Required Action |
|---------|--------|-----------------|
| `DB.prepareStatement()` followed by `DB.close(rs, pstmt)` | Conformant | None |
| `DB.getSQLValue*()` methods | Conformant | None (handles cleanup internally) |
| `DB.prepareStatement()` without `DB.close()` | Non-conformant | Add `DB.close(rs, pstmt)` |
| Manual `connection.prepareStatement()` | Non-conformant | Use `DB.prepareStatement()` instead |

**Conformant example:**
```groovy
def pstmt = DB.prepareStatement(sql, trxName)
pstmt.setInt(1, paramValue)
def rs = pstmt.executeQuery()
while (rs.next()) {
    // process rows
}
DB.close(rs, pstmt)  // Required cleanup
```

**Conformant alternative (convenience methods):**
```groovy
// These methods handle resource cleanup internally
def value = DB.getSQLValue(trxName, sql, param)
def valueBD = DB.getSQLValueBD(trxName, sql, param)
def valueString = DB.getSQLValueString(trxName, sql, param)
```

**Non-conformant example:**
```groovy
def pstmt = DB.prepareStatement(sql, trxName)
def rs = pstmt.executeQuery()
if (rs.next()) {
    return rs.getString(1)  // Early return without closing!
}
// Missing: DB.close(rs, pstmt)
```

## Adding New Checks

To add a new validation check:

1. Add a new `###` section under [Checks](#checks)
2. Document:
   - Issue description (what problem it prevents)
   - What to look for (patterns table)
   - Conformant and non-conformant examples
3. Update the TOC

**Future check candidates:**
- SQL injection in dynamic queries
- Hardcoded client/org IDs
- Missing transaction handling
- Deprecated API usage
