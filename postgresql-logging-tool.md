---
name: postgresql-logging
description: Enable and use PostgreSQL query logging to debug iDempiere issues including dynamic SQL analysis and query logic troubleshooting
compatibility: opencode
metadata:
  type: tool
  original_file: postgresql-logging-tool.md
  category: debugging
  scope: idempiere
---

# PostgreSQL Logging Tool

The purpose of this document is to describe how to enable and use PostgreSQL query logging to debug iDempiere issues.

This is important because iDempiere's info windows, processes, and reports generate SQL dynamically. When results are unexpected, viewing the actual SQL executed reveals context variable resolution issues, incorrect filters, and query logic problems.

## Quick Start

Enable logging, trigger the issue, check the log:

```bash
# Enable full SQL logging
incus exec <container> -- sudo -u postgres psql -c "ALTER SYSTEM SET log_statement = 'all';"
incus exec <container> -- sudo -u postgres psql -c "SELECT pg_reload_conf();"

# Trigger the issue in iDempiere UI

# Find and read the log (NixOS puts it in systemd private temp)
incus exec <container> -- sudo cat /var/lib/postgresql/*/data/current_logfiles
incus exec <container> -- sudo cat "<path-from-above>"
```

## Configuration

### Enable Logging

PostgreSQL logging requires two settings:

| Setting | Purpose | Requires Restart |
|---------|---------|------------------|
| `log_statement = 'all'` | Log all SQL statements | No (reload) |
| `logging_collector = 'on'` | Write logs to files | Yes (restart) |

```bash
# Enable statement logging (takes effect immediately after reload)
incus exec <container> -- sudo -u postgres psql -c "ALTER SYSTEM SET log_statement = 'all';"
incus exec <container> -- sudo -u postgres psql -c "SELECT pg_reload_conf();"

# Enable file logging (requires restart)
incus exec <container> -- sudo -u postgres psql -c "ALTER SYSTEM SET logging_collector = 'on';"
incus exec <container> -- sudo -u postgres psql -c "ALTER SYSTEM SET log_directory = '/tmp/pg_log';"
incus exec <container> -- sudo mkdir -p /tmp/pg_log && sudo chown postgres:postgres /tmp/pg_log
incus exec <container> -- sudo systemctl restart postgresql
```

### Verify Settings

```bash
incus exec <container> -- sudo -u postgres psql -c "SHOW log_statement; SHOW logging_collector; SHOW log_directory;"
```

### Disable Logging

```bash
incus exec <container> -- sudo -u postgres psql -c "ALTER SYSTEM SET log_statement = 'none';"
incus exec <container> -- sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

## Log File Location

### NixOS Containers

On NixOS, systemd's `PrivateTmp` isolates PostgreSQL's `/tmp`. The actual log path:

```
/tmp/systemd-private-<hash>-postgresql.service-<suffix>/tmp/pg_log/postgresql-<timestamp>.log
```

Find it via:

```bash
# Check current log file path
incus exec <container> -- sudo cat /var/lib/postgresql/*/data/current_logfiles

# Or search for it
incus exec <container> -- sudo find /tmp -name 'postgresql*.log' 2>/dev/null
```

### Without logging_collector

If `logging_collector = 'off'`, logs go to stderr/journald:

```bash
incus exec <container> -- sudo journalctl -u postgresql -n 100 --no-pager
```

## Reading Logs

### Filter for Specific Queries

```bash
# Find info window queries
incus exec <container> -- sudo cat "<log-path>" | grep -i "m_inout_createfrom_v" -A2

# Find queries with parameters
incus exec <container> -- sudo cat "<log-path>" | grep -E "(execute|DETAIL.*Parameters)"
```

### Log Format

Prepared statements show parameters separately:

```
[pid] LOG:  execute <name>: SELECT ... WHERE col = $1 AND col2 = $2
[pid] DETAIL:  Parameters: $1 = '1000208', $2 = 'N'
```

Key patterns:
- `execute <unnamed>`: Ad-hoc query
- `execute S_123`: Prepared statement (cached)
- `DETAIL: Parameters`: Actual parameter values

## Common Debugging Scenarios

### Info Window Returns No Results

1. Enable logging
2. Open the info window and trigger the query
3. Search log for the view name (e.g., `m_inout_createfrom_v`)
4. Check the WHERE clause parameters - often context variables (`@Variable@`) resolve incorrectly

### Context Variable Issues

iDempiere resolves `@Variable@` placeholders before executing SQL. Common problems:

| Symptom | Cause |
|---------|-------|
| Wrong ID value | Context reading from wrong window/tab level |
| NULL or 0 | Context variable not set |
| Org ID instead of record ID | Variable name collision |

The log shows the resolved value, not the placeholder, making it easy to spot misresolution.

> 💡 **Tip** - Compare the parameter value in the log against the expected value from the source record (e.g., check M_InOut.C_BPartner_ID vs what appears in the query).

## Related Documentation

- iDempiere Info Window Tool - Info window configuration and troubleshooting
- iDempiere Application Dictionary Tool - AD configuration reference

Tags: #tool-postgresql #tool-debug #idempiere
