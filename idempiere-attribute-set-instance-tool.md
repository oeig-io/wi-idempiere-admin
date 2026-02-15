---
name: idempiere-attribute-set-instance
description: Create and manage attribute sets in iDempiere for tracking product and inventory characteristics
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-attribute-set-instance-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Attribute Set Instance Tool

The purpose of this tool is to create and manage attribute sets in iDempiere for tracking product and inventory characteristics.

This is important because iDempiere supports two distinct attribute patterns that serve different business needs.

## TOC

- [AttributeValueType Reference](#attributevaluetype-reference)
- [Instance vs Product Attributes](#instance-vs-product-attributes)
- [Attribute Set Type](#attribute-set-type)
- [Creating Reference Attributes](#creating-reference-attributes)
- [Implementation Pattern](#implementation-pattern)
- [Key Tables Reference](#key-tables-reference)

## AttributeValueType Reference

The `m_attribute.attributevaluetype` column defines how attribute values are stored and presented.

| Code | Name | Use Case |
|------|------|----------|
| S | String (max 40) | Text values, simple identifiers |
| N | Number | Numeric measurements |
| L | List | Fixed options defined in attribute subtab |
| R | Reference | Dropdown pointing to existing table via ad_reference + ad_ref_table |
| D | Date | Date/time values |
| C | Chosen Multiple Selection List | Multi-select from fixed options |

### String (S)

Stores text up to 40 characters. Use for identifiers, codes, or short text values.

```sql
INSERT INTO m_attribute (
    m_attribute_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, attributevaluetype, isinstanceattribute, ismandatory, m_attribute_uu
) VALUES (
    nextval('m_attribute_sq'), v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Color', 'S', 'Y', 'N', uuid_generate_v4()
);
```

### Number (N)

Stores numeric values. Use for measurements, quantities, or dimensions.

```sql
INSERT INTO m_attribute (
    m_attribute_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, attributevaluetype, isinstanceattribute, ismandatory, m_attribute_uu
) VALUES (
    nextval('m_attribute_sq'), v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Length', 'N', 'Y', 'Y', uuid_generate_v4()
);
```

### List (L)

Allows users to define options directly in the attribute subtab. Each attribute instance can select from these options.

### Reference (R)

Creates a dropdown that pulls options from an existing iDempiere table. This requires using ad_reference with ad_ref_table to define the table relationship.

See [Creating Reference Attributes](#creating-reference-attributes) for details.

### Date (D)

Stores date/time values. Use for expiration dates, manufacturing dates, or other temporal attributes.

### Chosen Multiple Selection List (C)

Multi-select from a fixed set of options. Users can select multiple values from the list.

## Instance vs Product Attributes

iDempiere supports two fundamentally different attribute patterns:

| Aspect | Instance (isinstanceattribute=Y) | Product (isinstanceattribute=N) |
|--------|----------------------------------|--------------------------------|
| Storage | M_AttributeInstance | AD_TableAttribute |
| When set | At material receipt/transaction | At product definition |
| Uniqueness | Each inventory instance has unique values | All products of same type share values |
| Attribute Set Type | MMS (default) | MMS or TA (both work for product attributes) |

### Instance Attributes

Instance attributes track characteristics that vary per inventory instance. Each time material is received, unique values can be assigned.

**Example**: A stone slab has unique dimensions (Length, Width, Thickness) per slab.

```sql
INSERT INTO m_attribute (
    m_attribute_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, attributevaluetype, isinstanceattribute, ismandatory, m_attribute_uu
) VALUES (
    nextval('m_attribute_sq'), v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Block', 'S', 'Y', 'N', uuid_generate_v4()
);
-- isinstanceattribute = 'Y' marks this as an instance attribute
```

### Product Attributes

Product attributes apply to all products of the same type. The value is set once when the product is defined and applies to all inventory.

**Example**: A tile product has a Mat_Finish (Polished) that applies to all instances of that product.

```sql
INSERT INTO m_attribute (
    m_attribute_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, attributevaluetype, isinstanceattribute, ismandatory, m_attribute_uu
) VALUES (
    nextval('m_attribute_sq'), v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Mat_Finish', 'R', 'N', 'N', uuid_generate_v4()
);
-- isinstanceattribute = 'N' marks this as a product attribute
```

## Attribute Set Type

The `m_attributeset.m_attributeset_type` column controls how attribute values are stored:

| Type | Code | Description |
|------|------|-------------|
| Material Management System | MMS | Default, stores attribute values in M_AttributeInstance |
| Table Attribute | TA | Stores attribute values in AD_TableAttribute for product-level attributes |

### MMS (Default)

Use for instance attributes. Values stored in M_AttributeInstance per inventory transaction.

**Important:** Always explicitly set `islot` and `isserno` to 'N' unless the user explicitly requests lot or serial number tracking. This ensures tiles/slabs are not incorrectly marked as tracked inventory.

```sql
INSERT INTO m_attributeset (
    m_attributeset_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, description, isinstanceattribute,
    islot, islotmandatory, isserno, issernomandatory,
    isguaranteedate, isguaranteedatemandatory, guaranteedays,
    mandatorytype, m_attributeset_uu
) VALUES (
    nextval('m_attributeset_sq'), v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Stone Slab', 'Attributes for stone slabs', 'Y',
    'N', 'N', 'N', 'N',
    'N', 'N', 0,
    'N', uuid_generate_v4()
);
-- m_attributeset_type defaults to MMS
-- IMPORTANT: Always set islot='N' and isserno='N' unless explicitly requested
```

### Table Attribute (TA)

Use for product attributes. Values stored in AD_TableAttribute at the product level. Note: MMS also works for product attributes (used in Tile example).

**Important:** Even with TA type, explicitly set `islot` and `isserno` to 'N' unless the user explicitly requests lot or serial number tracking.

```sql
INSERT INTO m_attributeset (
    m_attributeset_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, description, isinstanceattribute,
    islot, islotmandatory, isserno, issernomandatory,
    m_attributeset_type, m_attributeset_uu
) VALUES (
    nextval('m_attributeset_sq'), v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Tile', 'Product attributes for tiles', 'N',
    'N', 'N', 'N', 'N',
    'TA', uuid_generate_v4()
);
```

## Creating Reference Attributes

Reference attributes create dropdowns that pull options from existing iDempiere tables. This uses the ad_reference + ad_ref_table pattern to define the table relationship.

### Step 1: Create AD_Reference with Table Validation

Create an ad_reference with validationtype = 'T' (Table Validation):

```sql
INSERT INTO ad_reference (
    ad_reference_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, validationtype, entitytype, ad_reference_uu
) VALUES (
    nextval('ad_reference_sq'), v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Mat_Type', 'T', 'U', uuid_generate_v4()
)
RETURNING ad_reference_id;
```

- `validationtype = 'T'` - Table Validation tells the system to use ad_ref_table for the dropdown

### Step 2: Create AD_Ref_Table

Link the ad_reference to the actual table:

```sql
INSERT INTO ad_ref_table (
    ad_reference_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    ad_table_id, ad_key, ad_display, whereclause, entitytype, ad_ref_table_uu
) VALUES (
    v_reference_id, v_client_id, 0, 'Y', now(), 100, now(), 100,
    (SELECT ad_table_id FROM ad_table WHERE tablename = 'ANS_Mat_Type'),
    (SELECT ad_column_id FROM ad_column WHERE ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'ANS_Mat_Type') AND columnname = 'ANS_Mat_Type_ID'),
    (SELECT ad_column_id FROM ad_column WHERE ad_table_id = (SELECT ad_table_id FROM ad_table WHERE tablename = 'ANS_Mat_Type') AND columnname = 'Name'),
    NULL,
    'U', uuid_generate_v4()
);
```

- `ad_table_id` - The custom table (ANS_Mat_Type)
- `ad_key` - The primary key column ID
- `ad_display` - The display name column ID

### Step 3: Create Attribute with Reference Type

Create the attribute pointing to the ad_reference:

```sql
INSERT INTO m_attribute (
    m_attribute_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, attributevaluetype, isinstanceattribute, ismandatory,
    ad_reference_id, ad_reference_value_id, m_attribute_uu
) VALUES (
    nextval('m_attribute_sq'), v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Mat_Type', 'R', 'N', 'N',
    18, v_reference_id, uuid_generate_v4()
);
```

Key fields:
- `attributevaluetype = 'R'` - Reference type
- `isinstanceattribute = 'N'` - Product attribute
- `ad_reference_id = 18` - Table display type
- `ad_reference_value_id` - Points to ad_reference

## Implementation Pattern

Complete implementation requires creating attributes, the attribute set, and linking them together:

### Step 1: Create Attributes

Create each m_attribute with appropriate type (S, N, L, R, D, C) and instance/product setting.

### Step 2: Create Attribute Set

Create m_attributeset with appropriate type (MMS or TA) and instance setting.

```sql
v_attributeset_id := nextval('m_attributeset_sq');
INSERT INTO m_attributeset (
    m_attributeset_id, ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    name, description, isinstanceattribute, m_attributeset_uu
) VALUES (
    v_attributeset_id, v_client_id, 0, 'Y', now(), 100, now(), 100,
    'Tile', 'Product attributes for tiles', 'N', uuid_generate_v4()
);
-- m_attributeset_type defaults to MMS
```

### Step 3: Link Attributes to Set

Use m_attributeuse to link attributes to the attribute set. The seqno controls display order.

```sql
INSERT INTO m_attributeuse (
    m_attributeset_id, m_attribute_id,
    ad_client_id, ad_org_id, isactive, created, createdby, updated, updatedby,
    seqno, m_attributeuse_uu
) VALUES (
    v_attributeset_id, v_attribute_id,
    v_client_id, 0, 'Y', now(), 100, now(), 100,
    10, uuid_generate_v4()
);
-- seqno = 10 determines display order (ascending)
```

## Key Tables Reference

| Table | Purpose |
|-------|---------|
| m_attribute | Attribute definitions (name, type, instance/product) |
| m_attributeset | Attribute set container (name, type) |
| m_attributeuse | Links attributes to attribute sets (seqno for ordering) |
| ad_reference | Defines dropdown sources (validationtype = T for table) |
| ad_ref_table | Links ad_reference to actual table (ad_table_id, ad_key, ad_display) |
| m_attributeinstance | Stores instance attribute values |
| ad_tableattribute | Stores product attribute values |

### Key Columns in m_attribute

| Column | Description |
|--------|-------------|
| name | Display name of the attribute |
| attributevaluetype | S, N, L, R, D, or C |
| isinstanceattribute | Y = instance, N = product |
| ismandatory | Y = required |
| ad_reference_id | Display type (use 18 for Table) |
| ad_reference_value_id | Points to ad_reference for dropdown source |
| m_attribute_uu | UUID for external references |

### Key Columns in m_attributeset

| Column | Description |
|--------|-------------|
| name | Display name of the attribute set |
| description | Description of purpose |
| isinstanceattribute | Y = instance, N = product |
| m_attributeset_type | MMS (default) or TA |

### Key Columns in m_attributeuse

| Column | Description |
|--------|-------------|
| m_attributeset_id | Parent attribute set |
| m_attribute_id | Linked attribute |
| seqno | Display order (lower numbers appear first) |

### Key Columns in ad_ref_table

| Column | Description |
|--------|-------------|
| ad_reference_id | Links to ad_reference |
| ad_table_id | The table providing dropdown options |
| ad_key | Primary key column (for storage) |
| ad_display | Display name column (for dropdown) |
| whereclause | Optional WHERE clause to filter options |

Tags: #tool #attribute-set #application-dictionary