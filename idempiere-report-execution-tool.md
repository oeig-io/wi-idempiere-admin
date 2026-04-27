---
name: idempiere-report-execution
description: Run iDempiere reports and print documents via REST API and visually verify output by converting PDF to images
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-report-execution-tool.md
  category: integration
  scope: idempiere
---

# iDempiere Report Execution Tool

The purpose of this tool is to run iDempiere reports and print documents programmatically via the REST API and visually verify the output.

This is important because AI agents need to generate reports, confirm layout changes, and verify print output without manual UI interaction. This enables a create-then-verify workflow for report customization.

## TOC

- [Concepts](#concepts)
- [Report Types](#report-types)
- [Run a Menu Report](#run-a-menu-report)
- [Print a Record](#print-a-record)
- [Save and View Report Output](#save-and-view-report-output)
- [Complete Examples](#complete-examples)
- [Troubleshooting](#troubleshooting)

## Concepts

iDempiere has two ways to produce printed output:

| Method | Description | REST Endpoint |
|--------|-------------|---------------|
| **Menu Report** | Report launched from the menu with parameters (e.g., Storage Detail) | `POST /api/v1/processes/{slug}` |
| **Record Print** | Print a specific record using its window print format (e.g., Sales Order) | `GET /api/v1/models/{table}/{id}/print` |

Both return the report as a **base64-encoded file** in the JSON response.

## Report Types

The `report-type` parameter controls the output format:

| Value | Format | Response Field |
|-------|--------|---------------|
| `PDF` (default) | PDF document | `reportFile` |
| `HTML` | HTML page | `exportFile` |
| `CSV` | Comma-separated values | `exportFile` |
| `XLS` | Excel 97-2003 | `exportFile` |
| `XLSX` | Excel 2007+ | `exportFile` |

> ⚠️ **Warning** - PDF reports return in `reportFile`. All other formats return in `exportFile`. Your parsing code must check both fields.

## Run a Menu Report

Menu reports are AD_Process records with `IsReport='Y'`. They are executed via the standard process endpoint.

### Discover Report Slug and Parameters

```bash
# Find the process slug
psqli -t -A -c "SELECT Slugify(value) FROM ad_process WHERE name = 'Storage Detail';"
# Returns: rv_storage

# Get report metadata including parameters via REST API
curl -sk "${API_URL}/processes/rv_storage" \
    -H "Authorization: Bearer $SESSION_TOKEN" | python3 -m json.tool
```

The response includes a `parameters` array with `parameterName` values for each parameter.

### Execute Report

```bash
# Run with no parameters (all defaults)
RESPONSE=$(curl -sk -X POST "${API_URL}/processes/${REPORT_SLUG}" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{"report-type": "PDF"}')

# Run with parameters
RESPONSE=$(curl -sk -X POST "${API_URL}/processes/rv_storage" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{
        "report-type": "PDF",
        "M_Warehouse_ID": 1000000
    }')
```

### Additional Report Parameters

| Parameter | Description |
|-----------|-------------|
| `report-type` | PDF, HTML, CSV, XLS, XLSX (default: PDF) |
| `is-summary` | Boolean — summary mode |
| `print-format-id` | Override the default print format (ID or UUID) |

### Response Structure

```json
{
    "AD_PInstance_ID": 1000073,
    "process": "rv_storage",
    "summary": "Report",
    "isError": false,
    "reportFile": "<base64-encoded-pdf>",
    "reportFileName": "Storage_Detail.pdf",
    "reportFileLength": 97256,
    "nodeId": "ae5c68bb-..."
}
```

For non-PDF formats, the fields are `exportFile`, `exportFileName`, and `exportFileLength` instead.

## Print a Record

Use the print endpoint to produce the same output as clicking the Print toolbar icon in a window.

```bash
# Print a Sales Order as PDF
curl -sk "${API_URL}/models/C_Order/${ORDER_ID}/print?\$report_type=PDF" \
    -H "Authorization: Bearer $SESSION_TOKEN"
```

> 📝 **Note** - The print endpoint requires that the window's header tab has an `AD_Process_ID` assigned (the print process). If no print process is defined, the API returns a 404 "No print process" error.

## Save and View Report Output

After receiving the REST API response, decode the base64 content, save it as a PDF, convert pages to images, and view them.

### Step 1: Decode and Save PDF

```bash
echo "$RESPONSE" | python3 -c "
import sys, json, base64
data = json.load(sys.stdin)
field = 'reportFile' if 'reportFile' in data else 'exportFile'
pdf_data = base64.b64decode(data[field])
with open('/tmp/report.pdf', 'wb') as f:
    f.write(pdf_data)
print('Saved:', data.get(field.replace('File','FileName'), 'report'))
print('Size:', len(pdf_data), 'bytes')
print('Error:', data.get('isError', False))
"
```

### Step 2: Convert PDF Pages to Images

Use `nix-shell` with `poppler-utils` to convert PDF pages to PNG:

```bash
# Convert all pages at 150 DPI
nix-shell -p poppler-utils --run \
    "pdftoppm -png -r 150 /tmp/report.pdf /tmp/report_page"

# Convert only first page (faster for quick checks)
nix-shell -p poppler-utils --run \
    "pdftoppm -png -r 150 -f 1 -l 1 /tmp/report.pdf /tmp/report_page"
```

This produces files like `/tmp/report_page-01.png`, `/tmp/report_page-02.png`, etc.

### Step 3: View the Image

Use the `read` tool to view the rendered page:

```
read /tmp/report_page-01.png
```

This allows visual verification of report layout, column headers, data content, and formatting.

## Complete Examples

### Menu Report (Storage Detail)

End-to-end: authenticate, run Storage Detail report, save PDF, convert to image, and view.

```bash
API_URL="https://localhost:9085/api/v1"

# Authenticate
AUTH_RESPONSE=$(curl -sk -X POST "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -d '{"userName":"SuperUser","password":"System"}')
TOKEN=$(echo "$AUTH_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)

# Select client context
SESSION_RESPONSE=$(curl -sk -X PUT "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"clientId":[CLIENT_ID],"roleId":[ROLE_ID],"organizationId":[ORG_ID],"warehouseId":0}')
SESSION_TOKEN=$(echo "$SESSION_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)

# Run report
RESPONSE=$(curl -sk -X POST "${API_URL}/processes/rv_storage" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $SESSION_TOKEN" \
    -d '{"report-type": "PDF"}')

# Save PDF
echo "$RESPONSE" | python3 -c "
import sys, json, base64
data = json.load(sys.stdin)
if data.get('isError'):
    print('ERROR:', data.get('summary'))
    sys.exit(1)
field = 'reportFile' if 'reportFile' in data else 'exportFile'
with open('/tmp/report.pdf', 'wb') as f:
    f.write(base64.b64decode(data[field]))
print('Saved:', data.get(field + 'Name'))
"

# Convert first page to image
nix-shell -p poppler-utils --run \
    "pdftoppm -png -r 150 -f 1 -l 1 /tmp/report.pdf /tmp/report_page"

# View
# Use: read /tmp/report_page-01.png
```

### Record Print (Sales Order)

End-to-end: authenticate, print a Sales Order, save PDF, convert to image, and view.

```bash
API_URL="https://localhost:9085/api/v1"

# Authenticate
AUTH_RESPONSE=$(curl -sk -X POST "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -d '{"userName":"SuperUser","password":"System"}')
TOKEN=$(echo "$AUTH_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)

# Select client context
SESSION_RESPONSE=$(curl -sk -X PUT "${API_URL}/auth/tokens" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"clientId":[CLIENT_ID],"roleId":[ROLE_ID],"organizationId":[ORG_ID],"warehouseId":0}')
SESSION_TOKEN=$(echo "$SESSION_RESPONSE" | grep -o '"token":"[^"]*"' | head -1 | cut -d'"' -f4)

# Find a Sales Order to print
ORDER_ID=$(psqli -t -A -c "SELECT c_order_id FROM c_order WHERE issotrx='Y' AND ad_client_id=[CLIENT_ID] ORDER BY c_order_id DESC LIMIT 1;")

# Print the record (GET, not POST)
RESPONSE=$(curl -sk "${API_URL}/models/C_Order/${ORDER_ID}/print?\$report_type=PDF" \
    -H "Authorization: Bearer $SESSION_TOKEN")

# Save PDF
echo "$RESPONSE" | python3 -c "
import sys, json, base64
data = json.load(sys.stdin)
if data.get('isError'):
    print('ERROR:', data.get('summary'))
    sys.exit(1)
field = 'reportFile' if 'reportFile' in data else 'exportFile'
with open('/tmp/sales_order.pdf', 'wb') as f:
    f.write(base64.b64decode(data[field]))
print('Saved:', data.get(field + 'Name'))
"

# Convert first page to image
nix-shell -p poppler-utils --run \
    "pdftoppm -png -r 150 -f 1 -l 1 /tmp/sales_order.pdf /tmp/so_page"

# View
# Use: read /tmp/so_page-1.png
```

## Troubleshooting

### 403 Access Denied for Process

The role does not have process access. Grant it:

```sql
INSERT INTO ad_process_access (ad_process_id, ad_role_id, ad_client_id, ad_org_id, isactive,
    created, createdby, updated, updatedby, isreadwrite, ad_process_access_uu)
SELECT p.ad_process_id, r.ad_role_id, r.ad_client_id, 0, 'Y',
    now(), 100, now(), 100, 'Y', uuid_generate_v4()
FROM ad_process p, ad_role r
WHERE p.value = '[PROCESS_VALUE]' AND r.name = '[ROLE_NAME]'
AND NOT EXISTS (
    SELECT 1 FROM ad_process_access pa
    WHERE pa.ad_process_id = p.ad_process_id AND pa.ad_role_id = r.ad_role_id
);
```

### 404 No Print Process

The window tab does not have a print process assigned. Check:

```sql
SELECT t.name, t.ad_process_id
FROM ad_tab t
JOIN ad_window w ON t.ad_window_id = w.ad_window_id
WHERE w.name = '[WINDOW_NAME]' AND t.tablevel = 0;
```

If `ad_process_id` is null, no print format is configured for that window.

### reportFile is Missing from Response

- For PDF: check `reportFile`
- For HTML/CSV/XLS/XLSX: check `exportFile`
- If both are missing, the process may have errored — check `isError` and `summary`

### nix-shell pdftoppm Not Found

Use the correct package name:

```bash
# Correct (hyphenated)
nix-shell -p poppler-utils --run "pdftoppm ..."

# Wrong (underscore) — will error
nix-shell -p poppler_utils --run "pdftoppm ..."
```

Tags: #tool #report #print #rest-api #idempiere
