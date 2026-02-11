---
name: idempiere-quick-info-widget
description: Configure the Quick Info Widget for contextual information display in the right-side help panel using AD_Message, AD_StatusLine, and AD_StatusLineUsedIn
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-quick-info-widget-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Quick Info Widget Tool

The Quick Info Widget displays contextual information in the right-side help panel. It is better than ColumnSQL because:

- Better optimized for performance
- Can show much more information
- Can be formatted with HTML to show status or conditions
- Can be accessed in both grid and detail views
- Groups can be expanded or minimized

For standard INSERT patterns and role access grants, see [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md).

## Overview

A Quick Info Widget requires three components:

| Component | Table | Purpose |
|-----------|-------|---------|
| Message | AD_Message | Template text with MessageFormat parameters |
| Status Line | AD_StatusLine | Links message to SQL query |
| Used In | AD_StatusLineUsedIn | Defines where the widget appears |

The `IsStatusLine` field in AD_StatusLineUsedIn determines display location:
- `Y` = Status bar at bottom of window
- `N` = Quick Info Widget in right-side help panel

## AD_Message

The message contains HTML and MessageFormat parameters. Parameters use `{n}` syntax (zero-indexed).

| Column | Value | Notes |
|--------|-------|-------|
| Value | Unique identifier | Used as lookup key |
| MsgType | `I` | Information message |
| MsgText | HTML with parameters | Supports `<br/>`, `<strong>`, `<em>`, etc. |

```sql
INSERT INTO ad_message (
    ad_message_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, value, msgtype, msgtext, entitytype, ad_message_uu
) VALUES (
    nextval('ad_message_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'ANS_QI_Related_Documents',
    'I',
    '<br /><strong>Related Shipments:</strong><br />
<em>(Move Date : Shipment # : Status)</em><br />{0}<br />
<br /><strong>Related Invoices:</strong><br />
<em>(Invoice Date : Invoice # : Status : Total : Paid?)</em><br />{1}',
    'U',
    uuid_generate_v4()::varchar
);
```

## AD_StatusLine

Links the message to an SQL query. The number of columns returned must match the number of parameters in the message.

| Column | Value | Notes |
|--------|-------|-------|
| Name | Descriptive name | Displayed in Status Line window |
| AD_Message_ID | Message ID | Links to AD_Message |
| SQLStatement | SQL query | Use context variables like `@C_Order_ID@` |

```sql
INSERT INTO ad_statusline (
    ad_statusline_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, ad_message_id, sqlstatement,
    entitytype, ad_statusline_uu
) VALUES (
    nextval('ad_statusline_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    'ANS_QI_Related_Documents',
    (SELECT ad_message_id FROM ad_message WHERE value = 'ANS_QI_Related_Documents'),
    'SELECT array_to_string(array(
        SELECT concat_ws('' '', to_char(s.MovementDate, ''MM-DD-YYYY''), s.DocumentNo, s.DocStatus)
        FROM M_InOut s
        WHERE s.C_Order_ID = @C_Order_ID@
        ORDER BY s.MovementDate DESC
    ), ''<br />''),
    array_to_string(array(
        SELECT concat_ws('' '', to_char(i.DateInvoiced, ''MM-DD-YYYY''), i.DocumentNo, i.DocStatus,
            to_char(i.GrandTotal, ''FM$9,999,999,990D00''), i.isPaid)
        FROM C_Invoice i
        WHERE i.C_Order_ID = @C_Order_ID@
        ORDER BY i.DateInvoiced DESC
    ), ''<br />'')',
    'U',
    uuid_generate_v4()::varchar
);
```

## AD_StatusLineUsedIn

Defines where the widget appears. Use one of three approaches:

| Approach | Fields Set | Use Case |
|----------|------------|----------|
| By Table | AD_Table_ID | Widget appears on all windows using this table |
| By Window | AD_Window_ID | Widget appears on all tabs in this window |
| By Window+Tab | AD_Window_ID, AD_Tab_ID | Widget appears only on specific tab |

```sql
-- Attach to Sales Order table (appears everywhere C_Order is used)
INSERT INTO ad_statuslineusedin (
    ad_statuslineusedin_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, ad_statusline_id, ad_table_id, isstatusline, seqno,
    entitytype, ad_statuslineusedin_uu
) VALUES (
    nextval('ad_statuslineusedin_sq'), 0, 0, 'Y', now(), 100, now(), 100,
    (SELECT ad_statusline_id FROM ad_statusline WHERE name = 'ANS_QI_Related_Documents'),
    259,  -- C_Order table
    'N',  -- N = Quick Info Widget (not status line)
    10,   -- Sequence for ordering multiple widgets
    'U',
    uuid_generate_v4()::varchar
);
```

## Context Variables

Use `@ColumnName@` syntax to access values from the current record:

| Pattern | Description |
|---------|-------------|
| `@C_Order_ID@` | Current order ID |
| `@C_BPartner_ID@` | Current business partner ID |
| `@M_Product_ID@` | Current product ID |
| `@#AD_Client_ID@` | Login client ID |
| `@#AD_Org_ID@` | Login org ID |

### Default Values for Null Variables

Use `@ColumnName:default@` syntax to provide a fallback when the variable is null or missing:

| Pattern | Description |
|---------|-------------|
| `@C_Project_ID:0@` | Project ID, defaults to 0 if null |
| `@M_Product_ID:-1@` | Product ID, defaults to -1 if null |

**Important:** Always use default values for optional fields. Without a default, null context variables create invalid SQL (e.g., `WHERE x = ` with nothing after the equals sign).

### Context Inheritance from Parent Tabs

Context variables inherit from parent tabs. On a Sales Order Line, `@C_Project_ID@` resolves to:
1. The line's C_Project_ID if set
2. Otherwise, the order header's C_Project_ID

This means a widget attached to C_OrderLine can access header fields through context inheritance.

### COALESCE for Header/Line Fields

When a field exists at both header and line level (like C_Project_ID), use COALESCE to let line-level values override:

```sql
-- Find all lines sharing the same effective project
WHERE COALESCE(ol.C_Project_ID, o.C_Project_ID) = @C_Project_ID:0@
```

This matches lines where:
- The line has its own project assignment matching the context, OR
- The line inherits from a header whose project matches the context

## SQL Patterns

### Multiple Rows as HTML List

Use `array_to_string(array(...), '<br />')` to convert multiple rows into a single HTML string:

```sql
SELECT array_to_string(array(
    SELECT concat_ws(' ',
        to_char(o.DateOrdered, 'MM-DD-YYYY'),
        o.DocumentNo,
        o.DocStatus,
        to_char(o.GrandTotal, 'FM$9,999,999,990D00'))
    FROM C_Order o
    WHERE o.C_BPartner_ID = @C_BPartner_ID@ AND o.IsSOTrx = 'Y'
    ORDER BY o.DateOrdered DESC
    FETCH FIRST 5 ROWS ONLY
), '<br />')
```

### Order Line Summary

```sql
SELECT array_to_string(array(
    SELECT concat_ws('-', ol.qtyEntered::integer, ol.qtyDelivered::integer, ol.qtyInvoiced::integer)
        || ' ' || concat_ws('...', left(p.name, 25), p.custom_size)
    FROM C_OrderLine ol
    LEFT OUTER JOIN m_product p ON p.m_product_id = ol.m_product_id
    WHERE ol.C_Order_ID = @C_Order_ID@
    ORDER BY ol.line
), '<br />')
```

## HTML Formatting

Basic HTML tags are supported in the message text:

| Tag | Purpose | Example |
|-----|---------|---------|
| `<br />` | Line break | Separate rows |
| `<strong>` | Bold | Section headers |
| `<em>` | Italic | Column descriptions |
| `<div style="...">` | Styling | Colors, alignment |

### Conditional Styling

Use MessageFormat choice syntax for conditional formatting:

```sql
-- In AD_Message.MsgText
<div style="{0,choice,-1#color:red;|0#color:gray;|0<color:green;}">{0,number,#.#}</div>
```

### Progress Bar

```html
<progress max="100" value="{0}"></progress>
```

## Dashboard Widget

To display a status line on the dashboard, create a PA_DashboardContent record:

```sql
INSERT INTO pa_dashboardcontent (
    pa_dashboardcontent_id, ad_client_id, ad_org_id, isactive, created, createdby,
    updated, updatedby, name, ad_statusline_id, columnno, line,
    iscollapsible, isshowintdashboard, isshowinlogin, isshowtitle,
    pa_dashboardcontent_uu
) VALUES (
    nextval('pa_dashboardcontent_sq'), 11, 0, 'Y', now(), 100, now(), 100,
    'Total Sales KPI',
    (SELECT ad_statusline_id FROM ad_statusline WHERE name = 'Dashboard_TotalSales'),
    1,    -- Column number
    10,   -- Line number (ordering)
    'Y',  -- Collapsible
    'Y',  -- Show in dashboard
    'Y',  -- Show in login
    'Y',  -- Show title
    uuid_generate_v4()::varchar
);
```

## Complete Example: Last 5 Orders Widget

```sql
DO $$
DECLARE
    v_message_id numeric;
    v_statusline_id numeric;
BEGIN
    -- Step 1: Create the message
    INSERT INTO ad_message (
        ad_message_id, ad_client_id, ad_org_id, isactive, created, createdby,
        updated, updatedby, value, msgtype, msgtext, entitytype, ad_message_uu
    ) VALUES (
        nextval('ad_message_sq'), 0, 0, 'Y', now(), 100, now(), 100,
        'ANS_QI_Last5Orders', 'I',
        '<br /><strong>Last 5 Orders For This BP:</strong><br />
<em>(Date : Order # : Status : Total)</em><br />{0}',
        'U', uuid_generate_v4()::varchar
    ) RETURNING ad_message_id INTO v_message_id;

    -- Step 2: Create the status line
    INSERT INTO ad_statusline (
        ad_statusline_id, ad_client_id, ad_org_id, isactive, created, createdby,
        updated, updatedby, name, ad_message_id, sqlstatement,
        entitytype, ad_statusline_uu
    ) VALUES (
        nextval('ad_statusline_sq'), 0, 0, 'Y', now(), 100, now(), 100,
        'ANS_QI_Last5Orders', v_message_id,
        'SELECT array_to_string(array(
            SELECT concat_ws('' '', to_char(o.DateOrdered, ''MM-DD-YYYY''), o.DocumentNo, o.DocStatus, to_char(o.GrandTotal, ''FM$9,999,999,990D00''))
            FROM C_Order o
            WHERE o.C_BPartner_ID = @C_BPartner_ID@ AND o.IsSOTrx = ''Y''
            ORDER BY o.DateOrdered DESC, o.c_order_id DESC
            FETCH FIRST 5 ROWS ONLY
        ), ''<br />'')',
        'U', uuid_generate_v4()::varchar
    ) RETURNING ad_statusline_id INTO v_statusline_id;

    -- Step 3: Attach to C_Order table as Quick Info Widget
    INSERT INTO ad_statuslineusedin (
        ad_statuslineusedin_id, ad_client_id, ad_org_id, isactive, created, createdby,
        updated, updatedby, ad_statusline_id, ad_table_id, isstatusline, seqno,
        entitytype, ad_statuslineusedin_uu
    ) VALUES (
        nextval('ad_statuslineusedin_sq'), 0, 0, 'Y', now(), 100, now(), 100,
        v_statusline_id,
        259,  -- C_Order table
        'N',  -- Quick Info Widget
        10,
        'U', uuid_generate_v4()::varchar
    );
END $$;
```

## Info Window Support

Quick Info Widgets can also be attached to Info Windows. Set AD_InfoWindow_ID instead of AD_Table_ID, and use special context variables:

| Variable | Description |
|----------|-------------|
| `@ColumnName@` | Parameter values (updated on Re-Query) |
| `@_IWInfo_ColumnName@` | Selected row values |
| `@_IWInfoIDs_Selected@` | Comma-separated IDs of selected rows |

```sql
-- For multi-select in Info Windows
WHERE C_Order_ID = ANY(string_to_array('@_IWInfoIDs_Selected@',',')::NUMERIC[])
```

## References

- [NF2.1 Quick Info Widget](http://wiki.idempiere.org/en/NF2.1_Quick_Info_Widget)
- [NF2.1 Configurable Status Line](https://wiki.idempiere.org/en/NF2.1_Configurable_Status_Line)
- [Status Line as Dashboard Widget](https://idempiere.org/blog/2024/07/10/status-line-as-dashboard-widget/)
- [NF11 Support Quick Info Widget for Info Windows](https://wiki.idempiere.org/en/NF11_Support_Quick_Info_Widget_for_Info_Windows)

Tags: #tool #idempiere #quick-info-widget #status-line
