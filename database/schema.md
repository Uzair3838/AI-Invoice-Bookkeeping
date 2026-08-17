# Data Table Schemas

This document describes the three n8n Data Tables used by the project.

> **Important:** These are application schemas, not exported runtime data. Do not commit actual invoice rows or error records to GitHub.

---

## 1. `pending_invoices`

### Purpose

Temporary storage for invoices that have passed the processing stage and are waiting for a human approval/rejection decision.

### Lifecycle

```text
Invoice processed
      |
      v
pending_invoices
      |
      +---- Approve ----> invoice_ledger + Google Sheets
      |
      +---- Reject -----> notification
      |
      v
Delete pending record
```

### Columns

| Column | Type | Description |
|---|---|---|
| `id` | Number / n8n row ID | Unique Data Table row identifier |
| `vendor` | String | Vendor/supplier name |
| `invoice_number` | String | Invoice identifier |
| `invoice_date` | String | Invoice date |
| `tax_id` | String | Vendor tax identifier |
| `currency` | String | Currency code |
| `subtotal` | Number | Invoice subtotal |
| `tax` | Number | Tax amount |
| `total` | Number | Final invoice total |
| `expense_category` | String | Categorized expense |
| `approval_status` | String | Pending approval state |
| `status` | String | Processing/business status |
| `created_at` | String / ISO timestamp | Time the pending record was created |
| `approved_at` | String / ISO timestamp | Approval timestamp when applicable |
| `review_reason` | String / nullable | Reason for manual review or rejection when applicable |

### Typical values

`approval_status`:

```text
PENDING
APPROVED
REJECTED
```

`status` may contain workflow-specific states such as:

```text
READY_FOR_APPROVAL
APPROVED
REJECTED
REVIEW_REQUIRED
```

---

## 2. `invoice_ledger`

### Purpose

Permanent internal bookkeeping/audit record for invoices that have completed the approval process.

The project documentation identifies the ledger fields as vendor, invoice number, invoice date, tax ID, currency, subtotal, tax, total, expense category, confidence, status, source file, and created timestamp. fileciteturn1file1L303-L328

### Columns

| Column | Type | Description |
|---|---|---|
| `id` | Number / n8n row ID | Unique Data Table row identifier |
| `vendor` | String | Vendor/supplier name |
| `invoice_number` | String | Invoice identifier |
| `invoice_date` | String | Invoice date |
| `tax_id` | String | Vendor tax identifier |
| `currency` | String | Currency code |
| `subtotal` | Number | Invoice subtotal |
| `tax` | Number | Tax amount |
| `total` | Number | Final invoice total |
| `expense_category` | String | Categorized expense |
| `confidence` | Number | AI extraction confidence |
| `status` | String | Final bookkeeping status |
| `source_file` | String | Original invoice file reference/name |
| `created_at` | String / ISO timestamp | Record creation timestamp |

### Typical status

```text
APPROVED
```

### Important distinction

`invoice_ledger` is the internal application ledger.

Google Sheets is a separate external bookkeeping/reporting destination. The project intentionally maintains both.

---

## 3. `invoice_error_log`

### Purpose

Persistent operational log for failures captured by the Error Handler workflow.

The table should contain enough information to diagnose a failed execution without requiring the original execution record to remain available indefinitely.

### Recommended columns

| Column | Type | Description |
|---|---|---|
| `id` | Number / n8n row ID | Unique Data Table row identifier |
| `timestamp` | String / ISO timestamp | When the error was recorded |
| `workflow_name` | String | Workflow that failed |
| `workflow_id` | String | n8n workflow identifier |
| `execution_id` | String | Failed execution identifier |
| `last_node_executed` | String | Last node reached before failure |
| `error_message` | String | Main error message |
| `error_details` | String / JSON text | Additional error information |
| `error_stack` | String / nullable | Stack trace when available |
| `execution_mode` | String / nullable | Manual, webhook, trigger, etc. |
| `status` | String | Error record status |

### Typical status

```text
ERROR
```

### Example conceptual record

```json
{
  "timestamp": "2026-08-17T12:00:00.000Z",
  "workflow_name": "Invoice Processing",
  "workflow_id": "example-workflow-id",
  "execution_id": "12345",
  "last_node_executed": "Basic LLM Chain",
  "error_message": "Request timed out",
  "error_details": "Model request exceeded timeout",
  "error_stack": null,
  "execution_mode": "manual",
  "status": "ERROR"
}
```

> **Implementation note:** Verify the exact columns currently configured in your `invoice_error_log` Data Table before treating this recommended schema as a literal export specification. The error handler's actual node mapping is the source of truth for the live table.

---

## Relationship Between Tables

There are no foreign-key constraints because these are n8n Data Tables, but the workflow establishes logical relationships.

```text
pending_invoices
      |
      | approval
      v
invoice_ledger

pending_invoices
      |
      | rejection
      v
deleted

Workflow failures
      |
      v
invoice_error_log
```

## Data Retention

### `pending_invoices`

Short-lived. A row exists only while an invoice is awaiting a decision or being handled.

### `invoice_ledger`

Long-lived. Approved invoice records are retained as internal bookkeeping/audit records.

### `invoice_error_log`

Long-lived operational history. Error records should be retained according to the project's operational/logging policy.

## GitHub Rule

Never commit the actual contents of these tables.

GitHub should contain only:
- schema documentation
- workflow definitions
- architecture documentation
- setup instructions

The live table data remains inside n8n.
