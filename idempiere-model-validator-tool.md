---
name: idempiere-model-validator
description: Groovy model validators for save events in iDempiere using Table Script Validator including AD_Rule and AD_Table_ScriptValidator configuration
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-model-validator-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Model Validator Tool

The purpose of this document is to provide patterns for creating Groovy model validators in iDempiere using Table Script Validator.

Model validators fire on table events (insert, update, delete) regardless of how the record is created (UI, REST API, import, or programmatic). This makes them essential for business logic that must always execute.

## When to Use Model Validators

Use model validators instead of callouts when:
- Logic must execute via REST API (callouts only fire in UI)
- Logic must execute during imports or batch processing
- Logic is core business rule, not just UI convenience

## Architecture

1. **AD_Rule** - Stores the Groovy script (eventtype = `T` for Table Event)
2. **AD_Table_ScriptValidator** - Links the rule to a table and event type

## Event Types

| Value | Name | When |
|-------|------|------|
| `TBN` | Table Before New | Before INSERT, can modify values |
| `TAN` | Table After New | After INSERT, record has ID |
| `TBC` | Table Before Change | Before UPDATE, can modify values |
| `TAC` | Table After Change | After UPDATE |
| `TBD` | Table Before Delete | Before DELETE, can prevent |
| `TAD` | Table After Delete | After DELETE |

For setting calculated fields, use `TBN` and `TBC` (before events allow modification).

## AD_Rule for Model Validator

| Column | Value | Notes |
|--------|-------|-------|
| value | `groovy:ScriptName` | Engine prefix required |
| eventtype | `T` | Table Event (not `P` for Process) |
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
    'groovy:OrderLineUOMConversion',
    'Order Line UOM Conversion',
    'Calculates QtyOrdered from QtyEntered and UOM conversion',
    'T', 'S',
    E'// Script content here - see examples below',
    'U', uuid_generate_v4()::varchar, '3'
);
```

## AD_Table_ScriptValidator

Links the rule to a specific table and event.

```sql
INSERT INTO ad_table_scriptvalidator (
    ad_table_scriptvalidator_id, ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby,
    ad_table_id, ad_rule_id, eventmodelvalidator, seqno,
    ad_table_scriptvalidator_uu
) VALUES (
    nextval('ad_table_scriptvalidator_sq'), 0, 0, 'Y',
    now(), 100, now(), 100,
    (SELECT ad_table_id FROM ad_table WHERE tablename = 'C_OrderLine'),
    (SELECT ad_rule_id FROM ad_rule WHERE value = 'groovy:OrderLineUOMConversion'),
    'TBN',  -- Table Before New
    10,
    uuid_generate_v4()::varchar
);

-- Also register for updates
INSERT INTO ad_table_scriptvalidator (
    ad_table_scriptvalidator_id, ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby,
    ad_table_id, ad_rule_id, eventmodelvalidator, seqno,
    ad_table_scriptvalidator_uu
) VALUES (
    nextval('ad_table_scriptvalidator_sq'), 0, 0, 'Y',
    now(), 100, now(), 100,
    (SELECT ad_table_id FROM ad_table WHERE tablename = 'C_OrderLine'),
    (SELECT ad_rule_id FROM ad_rule WHERE value = 'groovy:OrderLineUOMConversion'),
    'TBC',  -- Table Before Change
    10,
    uuid_generate_v4()::varchar
);
```

## Groovy Context Variables

Available in model validator scripts:

| Variable | Type | Description |
|----------|------|-------------|
| `A_Ctx` | Properties | Application context |
| `A_PO` | PO | The record being saved |
| `A_Type` | int | Event type constant |
| `A_Event` | String | Event name (e.g., "TBN") |

## Useful PO Methods

| Method | Description |
|--------|-------------|
| `A_PO.is_new()` | True if this is a new record (insert) |
| `A_PO.is_ValueChanged("ColumnName")` | True if the column value changed |
| `A_PO.get_Value("ColumnName")` | Get current value |
| `A_PO.get_ValueOld("ColumnName")` | Get previous value (before change) |
| `A_PO.set_ValueNoCheck("ColumnName", value)` | Set value without validation |

## Preventing Historical Data Contamination

When using TBC (Before Change), always check if relevant fields actually changed.
This prevents recalculating values when unrelated fields are edited:

```groovy
// Only process if this is new OR the relevant fields changed
if (!A_PO.is_new() && !A_PO.is_ValueChanged("QtyEntered") && !A_PO.is_ValueChanged("C_UOM_ID")) {
    return ""
}
// ... rest of calculation logic
```

## Return Values

- Return empty string `""` for success
- Return error message string to abort and show error
- Throwing an exception also aborts the save

## Example: Order Line UOM Conversion

This script replicates the callout logic that converts QtyEntered to QtyOrdered based on UOM:

```groovy
import org.compiere.model.MUOMConversion
import org.compiere.model.MProduct
import java.math.BigDecimal
import java.math.RoundingMode

// Get the order line PO
def ol = A_PO

// Only process if we have a product and UOM
def productId = ol.get_ValueAsInt("M_Product_ID")
def uomId = ol.get_ValueAsInt("C_UOM_ID")
def qtyEntered = ol.get_Value("QtyEntered") as BigDecimal

if (productId <= 0 || uomId <= 0 || qtyEntered == null) {
    return ""
}

// Get product's base UOM
def product = MProduct.get(A_Ctx, productId)
def productUomId = product.getC_UOM_ID()

// If UOMs are the same, QtyOrdered = QtyEntered
if (uomId == productUomId) {
    ol.set_ValueNoCheck("QtyOrdered", qtyEntered)
    return ""
}

// Convert from entered UOM to product UOM
def qtyOrdered = MUOMConversion.convertProductFrom(A_Ctx, productId, uomId, qtyEntered)

if (qtyOrdered == null) {
    // No conversion found, use QtyEntered as fallback
    qtyOrdered = qtyEntered
}

ol.set_ValueNoCheck("QtyOrdered", qtyOrdered)

// Also convert price if needed
def priceActual = ol.get_Value("PriceActual") as BigDecimal
if (priceActual != null && priceActual.signum() != 0) {
    def priceEntered = MUOMConversion.convertProductFrom(A_Ctx, productId, uomId, priceActual)
    if (priceEntered != null) {
        ol.set_ValueNoCheck("PriceEntered", priceEntered)
    }
}

return ""
```

## Testing

After creating the rule and validator:

1. Create a record via REST API
2. Verify the calculated fields are set correctly
3. Check iDempiere log for any script errors

## Debugging

Enable logging to see script execution:
```sql
-- Check if script fired
SELECT * FROM ad_changelog
WHERE ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'C_OrderLine')
ORDER BY created DESC LIMIT 10;
```

## Related Documentation

- [idempiere-callout-tool.md](idempiere-callout-tool.md) - Groovy callouts (eventtype = C, UI only)
- [idempiere-process-tool.md](idempiere-process-tool.md) - Groovy processes (eventtype = P)
- [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md) - SQL patterns

Tags: #tool #idempiere #model-validator #groovy
