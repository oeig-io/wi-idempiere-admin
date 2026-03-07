---
name: idempiere-process
description: Groovy processes implementation patterns for iDempiere including AD_Rule creation and AD_Process configuration with parameters
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-process-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Process Tool

The purpose of this document is to provide patterns for creating Groovy processes in iDempiere.

For standard INSERT patterns and role access grants, see [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md).

## AD_Rule

Stores the Groovy script.

| Column | Value | Notes |
|--------|-------|-------|
| value | `groovy:ScriptName` | Engine prefix required |
| eventtype | `P` | Process |
| ruletype | `S` | JSR223 Scripting |
| accesslevel | `3` | Client+Org (typical) |
| script | Groovy code | Return string message |

```sql
INSERT INTO ad_rule (
    ad_rule_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, value, name, description,
    eventtype, ruletype, script, entitytype, ad_rule_uu, accesslevel
) VALUES (
    nextval('ad_rule_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'groovy:ScriptName',
    'Process Display Name',
    'Description',
    'P', 'S',
    'return "Success message"',
    'U', uuid_generate_v4()::varchar, '3'
);
```

## AD_Process

References the AD_Rule via classname.

| Column | Value | Notes |
|--------|-------|-------|
| value | `ScriptName` | Unique identifier |
| classname | `@script:groovy:ScriptName` | Must match AD_Rule.value |
| allowmultipleexecution | `Y` | Required for Info Window processes |

```sql
INSERT INTO ad_process (
    ad_process_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, value, name, description,
    accesslevel, entitytype, isreport, isbetafunctionality,
    allowmultipleexecution, classname, ad_process_uu
) VALUES (
    nextval('ad_process_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'ScriptName',
    'Process Display Name',
    'Description',
    '3', 'U', 'N', 'N',
    'Y',
    '@script:groovy:ScriptName',
    uuid_generate_v4()::varchar
);
```

## Process Deployment Options

After creating the process, choose how users will access it:

| Deployment | Use When | Table(s) |
|------------|----------|----------|
| **Button** | Record-specific action (e.g., "Create PO from this SO") | AD_Column + AD_Field |
| **Info Window** | Multi-record selection with context | AD_InfoProcess |
| **Menu** | Standalone batch operation, no window context needed | AD_Menu + AD_TreeNodeMM |

### Decision Framework

| Choose Button when... | Choose Info Window when... | Choose Menu when... |
|----------------------|---------------------------|---------------------|
| User works in specific window | Need to select multiple records | Batch/administrative function |
| Current record provides context | Complex filtering/selection needed | No natural window context exists |
| Workflow-integrated action | Action applies to multiple items | All parameters entered at runtime |
| Single-record operation | Requires record selection UI | Self-contained business process |

> 🔗 **Reference**: See [idempiere-menu-tool.md](idempiere-menu-tool.md) for menu creation patterns.

## Linking to Info Window

AD_InfoProcess creates a button in an Info Window that runs the process.

```sql
INSERT INTO ad_infoprocess (
    ad_infoprocess_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, ad_infowindow_id, ad_process_id, seqno,
    layouttype, entitytype, ad_infoprocess_uu
) VALUES (
    nextval('ad_infoprocess_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    (SELECT ad_infowindow_id FROM ad_infowindow WHERE name = 'Window Name'),
    (SELECT ad_process_id FROM ad_process WHERE value = 'ScriptName'),
    10,
    'B',  -- Button
    'U', uuid_generate_v4()::varchar
);
```

## Groovy Context Variables

Available via ProcessUtil.java:

| Variable | Type | Description |
|----------|------|-------------|
| `A_Ctx` | Properties | Application context |
| `A_Trx` | Trx | Current transaction |
| `A_TrxName` | String | Transaction name |
| `A_Record_ID` | int | Current record ID |
| `A_AD_Client_ID` | int | Client ID |
| `A_AD_User_ID` | int | User ID |
| `A_AD_PInstance_ID` | int | Process instance ID |
| `A_Table_ID` | int | Table ID |
| `A_ProcessInfo` | ProcessInfo | Full process info object |
| `P_<paramname>` | varies | Process parameters |

## Return Values

- Return a string for success message
- Prefix with `@Error@` for errors: `return "@Error@Something failed"`

## Naming Convention

| Item | Convention | Example |
|------|------------|---------|
| AD_Rule.value | `groovy:ScriptName` | `groovy:SOToPOCreate` |
| AD_Process.classname | `@script:groovy:ScriptName` | `@script:groovy:SOToPOCreate` |

Tags: #tool #idempiere #process #groovy
