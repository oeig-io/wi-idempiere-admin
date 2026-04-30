---
name: idempiere-sales-tax
description: Sales tax rate lookup using SalesTaxZip API for iDempiere transactions
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-sales-tax-tool.md
  category: integration
  scope: idempiere
---

# iDempiere Sales Tax Tool

The purpose of this tool is to provide sales tax rate lookups by zip code for iDempiere transactions. This is important because iDempiere requires accurate tax rates for invoicing and compliance.

## TOC

- [Overview](#overview)
- [SalesTaxZip API](#salestaxzip-api)
- [iDempiere Integration](#idempiere-integration)
- [Limitations and TODOs](#limitations-and-todos)
- [Related Documentation](#related-documentation)

## Overview

iDempiere supports tax calculation, but the standard configuration requires pre-defined tax rates in the application dictionary. For dynamic tax rate lookup by zip code, we can integrate with the SalesTaxZip API.

**Use cases:**
- Look up combined sales tax rate for a given zip code
- Calculate tax amount on a transaction
- Reverse calculation: find pre-tax amount from total

## SalesTaxZip API

Free tier: 100 requests/hour, no API key required.

**Base URL:** `https://salestaxzip.com/api/v1`

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/rate/:zip` | Get tax rate by zip code |
| GET | `/state/:state` | List all zip codes and rates for a state |
| GET | `/city/:state/:city` | List zip codes and rates for a city |
| GET | `/reverse?total=X&zip=Y` | Reverse calculation (given total, find pre-tax) |

### Examples

```bash
# Get rate by zip (New York City)
curl https://salestaxzip.com/api/v1/rate/10001

# Response:
{
  "success": true,
  "data": {
    "zip_code": "10001",
    "city": "NEW YORK CITY",
    "state": "NY",
    "rates": {
      "combined": 0.08875,
      "combined_pct": "8.875%",
      "state": 0.04,
      "county": 0,
      "city": 0.045,
      "local": 0.048749999999999995
    },
    "calculation_examples": {
      "$100_purchase": { "tax": 8.88, "total": 108.88 }
    }
  }
}

# Reverse calculation
curl "https://salestaxzip.com/api/v1/reverse?total=108.88&zip=10001"

# Response:
{
  "success": true,
  "data": {
    "total_price": 108.88,
    "pre_tax_price": 100,
    "tax_amount": 8.88,
    "tax_rate": 0.08875
  }
}
```

### Rate Response Fields

| Field | Description |
|-------|-------------|
| `zip_code` | 5-digit US zip code |
| `city` | City name |
| `state` | 2-letter state code |
| `rates.combined` | Total tax rate (decimal) |
| `rates.state` | State tax rate |
| `rates.county` | County tax rate |
| `rates.city` | City tax rate |
| `rates.local` | Local/special district rate |

## iDempiere Integration

> 💬 **Comment** - Integration patterns are not yet documented.

Possible integration approaches:

1. **Callout on Business Partner address** - Auto-populate tax rate when zip changes
2. **Process or Groovy script** - Calculate tax during order/invoice creation
3. **Model Validator** - Validate tax rates against API before document completion
4. **External service integration** - Formal integration via iDempiere's service framework

### Business Partner Fields Needed

To look up tax by zip, we need:
- The postal code from the BP ship-to or bill-to address (`C_Location.Postal`, joined via `C_BPartner_Location.C_Location_ID`)

### Tax Configuration Considerations

- iDempiere requires `C_Tax` records for each tax rate
- The API returns the combined rate, but iDempiere may need component breakdown
- Consider creating tax rates dynamically or maintaining a mapping table

## Limitations and TODOs

- [ ] Document how to create a Groovy callout for tax rate lookup
- [ ] Document how to create a process for bulk tax rate updates
- [ ] Determine how to handle multi-component tax rates (state + county + city)
- [ ] Consider rate caching strategy to stay within 100 req/hour limit
- [ ] Evaluate accuracy against known tax rates (audit against Avalara or state sources)

### Known Limitations

1. **Rate limit**: 100 requests/hour may not be sufficient for high-volume scenarios
2. **Tax exemption**: API doesn't handle exemption certificates
3. **Product-specific rates**: API returns general rates, may not account for product taxability
4. **Boundary zones**: Zip codes on county borders may need special handling

## Related Documentation

- [SalesTaxZip API Docs](https://salestaxzip.com/api)
- [idempiere-callout-tool.md](idempiere-callout-tool.md) - For implementing tax lookup on field change
- [idempiere-process-tool.md](idempiere-process-tool.md) - For batch tax operations
- [Sales Tax API Research](../corporate/planning/sales-tax-api-discussion.md)

---

**Tags:** #tool #integration #sales-tax
