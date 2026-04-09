---
name: idempiere-callout
description: Groovy callouts for UI field change events in iDempiere including AD_Rule creation and AD_Column.Callout configuration
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-callout-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Callout Tool

The purpose of this document is to provide patterns for creating Groovy callouts in iDempiere.

Callouts fire immediately when a user changes a field value in the UI, providing instant feedback before the record is saved. This makes them ideal for UI convenience features like auto-populating related fields.

> 🔗 **Deployment:** All Groovy deploy scripts must use the `.sh` + temp file pattern. See [idempiere-groovy-deploy-pattern-tool.md](idempiere-groovy-deploy-pattern-tool.md) for the standard deployment approach.

## When to Use Callouts

Use callouts when:
- Logic should provide immediate UI feedback on field change
- Logic is UI convenience, not core business rule
- User needs to see calculated values before saving

Use model validators instead when:
- Logic must execute via REST API (callouts only fire in UI)
- Logic must execute during imports or batch processing
- Logic is a core business rule that must always execute

> 🔗 **Reference** - See [idempiere-model-validator-tool.md](idempiere-model-validator-tool.md) for model validators

## Architecture

1. **AD_Rule** - Stores the Groovy script (eventtype = `C` for Callout)
2. **AD_Column.Callout** - Links the rule to a column via `@script:groovy:RuleName`

## AD_Rule for Callout

| Column | Value | Notes |
|--------|-------|-------|
| value | `groovy:ScriptName` | Engine prefix required |
| eventtype | `C` | Callout (not `T` for Table Event) |
| ruletype | `S` | JSR223 Scripting |
| accesslevel | `3` | Client+Org (typical) |
| script | Groovy code | Return empty string for success |

```sql
INSERT INTO ad_rule (
    ad_rule_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, value, name, description,
    eventtype, ruletype, script, entitytype, ad_rule_uu, accesslevel
) VALUES (
    nextval('ad_rule_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'groovy:OrderLineASIQtyCallout',
    'Order Line ASI Quantity Callout',
    'Sets QtyEntered from storage when ANS_AttributeSetInstance_ID is selected.',
    'C', 'S',
    E'// Script content here - see examples below',
    'U', uuid_generate_v4(), '3'
);
```

## Linking Callout to Column

Set the `callout` field on AD_Column to reference the rule:

```sql
UPDATE ad_column
SET callout = '@script:groovy:OrderLineASIQtyCallout',
    updated = now()
WHERE columnname = 'ANS_AttributeSetInstance_ID'
  AND ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'C_OrderLine');
```

The format is `@script:groovy:RuleName` where `RuleName` matches the AD_Rule.value (without the `groovy:` prefix in the reference).

## Groovy Context Variables

Available in callout scripts:

| Variable | Type | Description |
|----------|------|-------------|
| `A_WindowNo` | int | Window number |
| `A_Tab` | GridTab | The tab containing the record |
| `A_Field` | GridField | The field that triggered the callout |
| `A_Value` | Object | New value of the field |
| `A_OldValue` | Object | Previous value of the field |
| `A_Ctx` | Properties | Application context |

## Useful GridTab Methods

| Method | Description |
|--------|-------------|
| `A_Tab.setValue("ColumnName", value)` | Set field value (updates UI immediately) |
| `A_Tab.getValue("ColumnName")` | Get current field value |
| `A_Tab.getValueAsInt("ColumnName")` | Get field value as integer |

## Return Values

- Return empty string `""` for success
- Return error message string to show error to user
- Errors do not prevent the field change, just display a message

## Example: ASI Quantity Lookup

This callout sets QtyEntered when the user selects an Attribute Set Instance:

```groovy
import org.compiere.util.DB
import java.math.BigDecimal

// A_Value is the new ANS_AttributeSetInstance_ID value
def asiId = A_Value as Integer

// Skip if no ASI selected
if (asiId == null || asiId <= 0) {
    return ""
}

// Look up total qty on hand from storage for this ASI
def sql = """
    SELECT SUM(soh.QtyOnHand) as TotalQty
    FROM M_StorageOnHand soh
    WHERE soh.M_AttributeSetInstance_ID = ?
      AND soh.QtyOnHand > 0
"""
def qtyOnHand = DB.getSQLValueBD(null, sql, asiId)

if (qtyOnHand == null || qtyOnHand.signum() <= 0) {
    return ""
}

// Set QtyEntered on the tab - this updates the UI immediately
A_Tab.setValue("QtyEntered", qtyOnHand)

return ""
```

## Complete SQL Example

Creates a callout and links it to a column:

```sql
DO $$
DECLARE
    v_rule_id INTEGER;
    v_column_id INTEGER;
    v_script TEXT;
BEGIN
    -- Get the target column
    SELECT ad_column_id INTO v_column_id
    FROM ad_column
    WHERE columnname = 'ANS_AttributeSetInstance_ID'
      AND ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'C_OrderLine');

    IF v_column_id IS NULL THEN
        RAISE NOTICE 'Column not found. Skipping.';
        RETURN;
    END IF;

    -- Check if rule already exists
    SELECT ad_rule_id INTO v_rule_id
    FROM ad_rule WHERE value = 'groovy:OrderLineASIQtyCallout';

    -- The Groovy callout script
    v_script := '
import org.compiere.util.DB
import java.math.BigDecimal

def asiId = A_Value as Integer
if (asiId == null || asiId <= 0) {
    return ""
}

def sql = """
    SELECT SUM(soh.QtyOnHand) as TotalQty
    FROM M_StorageOnHand soh
    WHERE soh.M_AttributeSetInstance_ID = ?
      AND soh.QtyOnHand > 0
"""
def qtyOnHand = DB.getSQLValueBD(null, sql, asiId)

if (qtyOnHand == null || qtyOnHand.signum() <= 0) {
    return ""
}

A_Tab.setValue("QtyEntered", qtyOnHand)
return ""
';

    IF v_rule_id IS NULL THEN
        -- Create the rule
        v_rule_id := nextval('ad_rule_sq');
        INSERT INTO ad_rule (
            ad_rule_id, ad_client_id, ad_org_id, isactive, created, createdby,
            updated, updatedby, value, name, description,
            eventtype, ruletype, script, entitytype, ad_rule_uu, accesslevel
        ) VALUES (
            v_rule_id, 0, 0, 'Y', now(), 100, now(), 100,
            'groovy:OrderLineASIQtyCallout',
            'Order Line ASI Quantity Callout',
            'Sets QtyEntered from storage when ANS_AttributeSetInstance_ID is selected.',
            'C', 'S', v_script, 'U', uuid_generate_v4(), '3'
        );
        RAISE NOTICE 'Created AD_Rule (ID: %)', v_rule_id;
    ELSE
        -- Update existing rule
        UPDATE ad_rule SET script = v_script, updated = now() WHERE ad_rule_id = v_rule_id;
        RAISE NOTICE 'Updated AD_Rule (ID: %)', v_rule_id;
    END IF;

    -- Link callout to column
    UPDATE ad_column
    SET callout = '@script:groovy:OrderLineASIQtyCallout',
        updated = now()
    WHERE ad_column_id = v_column_id
      AND (callout IS NULL OR callout = '' OR callout NOT LIKE '%OrderLineASIQtyCallout%');

    RAISE NOTICE 'Callout setup complete';
END $$;
```

## Multiple Callouts on One Column

If a column already has a callout, append the new one with a semicolon:

```sql
UPDATE ad_column
SET callout = CASE
    WHEN callout IS NULL OR callout = '' THEN '@script:groovy:NewCallout'
    ELSE callout || ';@script:groovy:NewCallout'
END
WHERE ad_column_id = ?;
```

## Testing

After creating the callout:

1. Open the window in iDempiere UI
2. Change the field value
3. Verify the target fields update immediately
4. Check iDempiere log for script errors if it doesn't work

## Debugging

Check iDempiere server log for script errors:
```bash
tail -f /opt/idempiere-server/log/idempiere.*.log | grep -i callout
```

## Callout vs Model Validator Comparison

| Aspect | Callout | Model Validator |
|--------|---------|-----------------|
| When fires | Field change in UI | Record save |
| REST API | No | Yes |
| Import/Batch | No | Yes |
| User feedback | Immediate | After save |
| Use case | UI convenience | Business rules |
| EventType | `C` | `T` |
| Linked via | AD_Column.Callout | AD_Table_ScriptValidator |

## Related Documentation

- [idempiere-model-validator-tool.md](idempiere-model-validator-tool.md) - Groovy model validators (eventtype = T)
- [idempiere-process-tool.md](idempiere-process-tool.md) - Groovy processes (eventtype = P)
- [idempiere-groovy-deploy-pattern-tool.md](idempiere-groovy-deploy-pattern-tool.md) - Standard pattern for deploying Groovy to AD_Rule
- [Script Callout - iDempiere Wiki](https://wiki.idempiere.org/en/Script_Callout)

Tags: #tool #idempiere #callout #groovy
