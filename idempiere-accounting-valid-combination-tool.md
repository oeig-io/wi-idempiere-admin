---
name: idempiere-accounting-valid-combination
description: Understanding and creating GL account valid combinations for charges, products, business partners, and other entities in iDempiere
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-accounting-valid-combination-tool.md
  category: accounting
  scope: idempiere
---

# iDempiere Accounting Valid Combination Tool

The purpose of this document is to explain how C_ValidCombination records work in iDempiere and how to create them when needed for proper accounting.

This is important because charges, products, business partners, and other entities cannot post directly to GL accounts. Instead, they post through valid combinations that include the GL account plus optional dimensional tracking (organization, product, project, business partner, etc.).

## What Are Valid Combinations?

A valid combination is a unique pairing of a GL account (C_ElementValue) with zero or more accounting dimensions. Think of it as a fully-qualified accounting destination.

### Relationship Chain

```
Transaction Entity (Charge/Product/BPartner)
    ↓
C_[Entity]_Acct (e.g., C_Charge_Acct, M_Product_Acct, C_BP_Group_Acct)
    ↓
Ch_Expense_Acct / P_Expense_Acct / etc. → C_ValidCombination.C_ValidCombination_ID
                                            ↓
                                      C_ValidCombination.Account_ID → C_ElementValue.C_ElementValue_ID
```

**Key insight:** You never post directly to a GL account. You always post through a valid combination. This allows iDempiere to track transactions by account AND by dimensions like product, project, business partner, sales region, etc.

### When Valid Combinations Are Created

iDempiere **automatically creates** valid combinations in many scenarios:
- When you use a GL account in a journal entry
- When you use a GL account in a charge, product, or BP group
- When you use a GL account in a document type

However, you may need to **manually create** valid combinations when:
- Setting up charges with specific GL accounts before any transactions occur
- Configuring products or BP groups with accounts that haven't been used yet
- Creating accounting schemas with dimensional requirements

## Valid Combination Naming Pattern

### Combination Format

```
[ORG]-[GLACCOUNT]-[PRODUCT]-[BPARTNER]-[PROJECT]-[CAMPAIGN]-[ACTIVITY]
```

**Simplified format (most common):**
```
*-GLACCOUNT-_-_-_-_-_
```

**Components:**
- `*` = wildcard (any organization, or organization not specified)
- `GLACCOUNT` = the GL account value (e.g., 54000)
- `-` = separator
- `_` = wildcard for optional dimension not specified

### Description Format

```
*-AccountName-_-_-_-_-_
```

Match the description to the account name for clarity. Example:
- Combination: `*-54000-_-_-_-_-_`
- Description: `*-Freight in-_-_-_-_-_`

## Creating Valid Combinations

### Check If One Exists

Before creating, always check if a valid combination already exists:

```sql
-- Check for account 54000 with no dimensions
SELECT vc.c_validcombination_id, vc.combination, vc.description
FROM c_validcombination vc
JOIN c_elementvalue ev ON vc.account_id = ev.c_elementvalue_id
WHERE vc.ad_client_id = 1000000
  AND ev.value = '54000'
  AND vc.c_acctschema_id = 1000000
  AND vc.isactive = 'Y'
  AND vc.m_product_id IS NULL
  AND vc.c_bpartner_id IS NULL
  AND vc.c_project_id IS NULL
  AND vc.c_campaign_id IS NULL
  AND vc.c_activity_id IS NULL;
```

### Create Valid Combination

If not found, create it:

```sql
-- Get next ID from sequence
SELECT nextval('c_validcombination_sq') INTO v_valid_comb_id;

-- Create valid combination
INSERT INTO c_validcombination (
    c_validcombination_id, ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby,
    c_acctschema_id, account_id,
    combination, description, isfullyqualified,
    c_validcombination_uu
) VALUES (
    v_valid_comb_id, 1000000, 0, 'Y',
    now(), 100, now(), 100,
    1000000, (SELECT c_elementvalue_id FROM c_elementvalue WHERE value = '54000'),
    '*-54000-_-_-_-_-_', '*-Freight in-_-_-_-_-_', 'Y',
    gen_random_uuid()
);
```

## Example: Applying Valid Combinations to Charges

This section demonstrates the complete pattern using charges as an example. The same principles apply to products, business partners, and other entities.

### The Charge Use Case

You want to create a "Pallet Fee" charge that posts to GL account 54000 "Freight in" instead of the default "Charge expense" account.

### Prerequisites

- Account 54000 exists in C_ElementValue
- You know the accounting schema ID (typically matches your client ID)

### Step 1: Create Valid Combination (If Needed)

See "Creating Valid Combinations" section above. Verify this returns a result:

```sql
SELECT vc.c_validcombination_id 
FROM c_validcombination vc
JOIN c_elementvalue ev ON vc.account_id = ev.c_elementvalue_id
WHERE vc.ad_client_id = 1000000 
  AND ev.value = '54000'
  AND vc.isactive = 'Y'
  AND vc.m_product_id IS NULL AND vc.c_bpartner_id IS NULL;
```

### Step 2: Create Charge via REST API

This creates the charge. iDempiere will auto-create C_Charge_Acct with a default expense account.

```bash
# Authenticate (see idempiere-rest-api-tool.md)
AUTH_RESPONSE=$(curl -s -X POST "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -d "{\"userName\":\"${ADMIN_USER}\",\"password\":\"${ADMIN_PASSWORD}\"}")

# Create charge
RESPONSE=$(curl -s -X POST "${API_URL}/models/C_Charge" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{
        "Name": "Pallet Fee",
        "ChargeAmt": 0,
        "C_TaxCategory_ID": 1000000,
        "IsSameTax": false,
        "IsSameCurrency": false
    }')

CHARGE_ID=$(echo "$RESPONSE" | grep -o '"id":[0-9]*' | head -1 | cut -d':' -f2)
```

### Step 3: Link Charge to Valid Combination

Now update C_Charge_Acct to use your valid combination:

```sql
-- Update to use your valid combination
UPDATE c_charge_acct 
SET ch_expense_acct = 1000054,  -- Your valid combination ID from Step 1
    updated = now(),
    updatedby = 100
WHERE c_charge_id = 1000001      -- Charge ID from Step 2
  AND c_acctschema_id = 1000000; -- Your accounting schema
```

### Step 4: Verify

```sql
-- Verify charge and accounting
SELECT 
    c.name as charge_name,
    c.chargeamt,
    vc.combination,
    ev.value as gl_account,
    ev.name as account_name
FROM c_charge c
JOIN c_charge_acct ca ON c.c_charge_id = ca.c_charge_id
JOIN c_validcombination vc ON ca.ch_expense_acct = vc.c_validcombination_id
JOIN c_elementvalue ev ON vc.account_id = ev.c_elementvalue_id
WHERE c.c_charge_id = 1000001;
```

Result:
```
 charge_name | chargeamt |    combination    | gl_account | account_name 
-------------+-----------+-------------------+------------+--------------
 Pallet Fee  |         0 | *-54000-_-_-_-_-_ | 54000      | Freight in
```

## Common Use Cases

### Use Case 1: Freight/Handling Charges (Charges)

**Scenario:** Add pallet fees, freight charges, or handling fees to orders.

**Account:** 54000 "Freight in" (expense account for shipping costs)

**Pattern:** Charge → C_Charge_Acct → Valid Combination (54000) → Freight in account

### Use Case 2: Product COGS

**Scenario:** Track cost of goods sold for products.

**Account:** 51100 "Product CoGs" (or appropriate COGS account)

**Pattern:** Product → M_Product_Acct → Valid Combination (51100) → Product CoGs account

> 🔗 **Reference:** See idempiere-product-tool.md for product accounting details

### Use Case 3: Business Partner Receivables/Payables

**Scenario:** Track A/R and A/P by business partner group.

**Account:** 12100 "Accounts Receivable - Trade" or 21100 "Accounts Payable"

**Pattern:** BP Group → C_BP_Group_Acct → Valid Combination (12100/21100) → Receivables/Payables account

> 🔗 **Reference:** See idempiere-bpartner-tool.md for BP accounting details

### Use Case 4: Document Type Accounting

**Scenario:** Configure different accounts for different document types (e.g., sales orders vs credit memos).

**Account:** Varies by document type purpose

**Pattern:** Document Type → C_DocType_Acct → Valid Combination → GL account

## Troubleshooting

### Issue: Valid combination doesn't exist for account

**Symptom:** You need to assign GL account 54000 to a charge, but no valid combination exists.

**Solution:** Create the valid combination manually as shown in "Creating Valid Combinations" above. Check:
- Account exists in C_ElementValue
- Accounting schema is correct
- No other valid combination conflicts

### Issue: Cannot find [Entity]_Acct record

**Symptom:** After creating entity (charge, product), cannot find the accounting record.

**Cause:** iDempiere creates [Entity]_Acct records asynchronously or during save validation.

**Solution:**
1. Wait 5-10 seconds after entity creation
2. Check: `SELECT * FROM c_charge_acct WHERE c_charge_id = ?`
3. If missing, check AD_Changelog for validation errors

### Issue: Wrong account posting

**Symptom:** Transaction posts to default "Charge expense" instead of desired account.

**Cause:** C_[Entity]_Acct not updated to point to correct valid combination.

**Solution:**
1. Find current valid combination: `SELECT ch_expense_acct FROM c_charge_acct WHERE c_charge_id = ?`
2. Update to correct valid combination ID
3. Re-run transaction

### Issue: Multiple accounting schemas

**Symptom:** C_[Entity]_Acct update affects wrong schema or multiple records exist.

**Cause:** Multi-schema setups have one [Entity]_Acct record per schema.

**Solution:** Always include `c_acctschema_id` in your WHERE clause:
```sql
UPDATE c_charge_acct 
SET ch_expense_acct = ?
WHERE c_charge_id = ? 
  AND c_acctschema_id = ?;  -- Always specify schema
```

## Reference

**Core Tables:**
- **C_ElementValue:** GL accounts (the "what")
- **C_ValidCombination:** GL account + dimensions (the "where with context")
- **C_[Entity]_Acct:** Links entity to valid combination (the "who posts where")

**Entity-Specific Accounting Tables:**
- C_Charge_Acct: Charge accounting (Ch_Expense_Acct, Ch_Revenue_Acct)
- M_Product_Acct: Product accounting (P_Asset_Acct, P_Expense_Acct, etc.)
- C_BP_Group_Acct: BP group accounting (C_Receivable_Acct, V_Liability_Acct)
- C_DocType_Acct: Document type accounting
- C_Tax_Acct: Tax accounting

**Key Principle:** Every accounting posting flows through C_ValidCombination. Never post directly to C_ElementValue.

Tags: #tool #accounting #gl #valid-combination #charge #product #bpartner #idempiere
