# iDempiere Enhancement

Work instructions for planning and executing iDempiere changes.

## Work Instructions

### Roles

- [idempiere-system-admin-role.md](idempiere-system-admin-role.md) - tenant configuration and go-live data

### Tasks

- [idempiere-environment-management-task.md](idempiere-environment-management-task.md) - incus environment workflow
- [idempiere-feature-configuration-task.md](idempiere-feature-configuration-task.md) - creating and testing configuration artifacts

### Tools

- [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md) - AD foundations: standard patterns, button columns, reference types
  - [idempiere-pre-minted-uuid-tool.md](idempiere-pre-minted-uuid-tool.md) - pre-mint UUIDs for stable cross-script entity references
  - [idempiere-table-create-tool.md](idempiere-table-create-tool.md) - creating new tables via UI
  - [idempiere-column-create-tool.md](idempiere-column-create-tool.md) - adding columns to existing tables (UI and SQL)
  - [idempiere-callout-tool.md](idempiere-callout-tool.md) - Groovy callouts (AD_Rule, AD_Column.Callout) - UI field change events
  - [idempiere-model-validator-tool.md](idempiere-model-validator-tool.md) - Groovy model validators (AD_Rule, AD_Table_ScriptValidator) - save events
  - [idempiere-process-tool.md](idempiere-process-tool.md) - Groovy processes (AD_Rule, AD_Process)
  - [idempiere-info-window-tool.md](idempiere-info-window-tool.md) - Info Windows (AD_InfoWindow, AD_InfoColumn)
- [idempiere-deploy-sql-tool.md](idempiere-deploy-sql-tool.md) - conventions for portable SQL deploy scripts (variable substitution, dynamic lookups, .sql vs .sh)
- [idempiere-code-validator-tool.md](idempiere-code-validator-tool.md) - audit Groovy/SQL for resource leaks and conventions
- [idempiere-pull-request-tool.md](idempiere-pull-request-tool.md) - checklist and workflow for submitting PRs to iDempiere core
- [idempiere-packout-tool.md](idempiere-packout-tool.md) - PackOut (2Pack) export records for portable artifact packaging
- [idempiere-rest-api-tool.md](idempiere-rest-api-tool.md) - REST API patterns for authentication, CRUD, processes
- [postgresql-logging-tool.md](postgresql-logging-tool.md) - PostgreSQL query logging for debugging iDempiere
- [ssh-remote-connection-tool.md](ssh-remote-connection-tool.md) - SSH key creation and remote access
