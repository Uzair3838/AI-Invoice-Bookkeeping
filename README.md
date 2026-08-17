# AI Invoice & Bookkeeping Automation

An n8n-based invoice-processing and bookkeeping automation system that extracts structured information from PDF invoices, validates the extracted values, detects duplicates, stores invoices pending human approval, and finalizes approved invoices into an internal ledger and Google Sheets.

## Project Status

**Status:** Completed working prototype

The current implementation is intentionally focused on **PDF invoices**. Image/OCR ingestion is outside the final project scope.

## Key Capabilities

- PDF invoice text extraction
- AI-powered invoice data structuring using an Ollama model
- Strict JSON parsing
- Deterministic invoice validation
- Line-item arithmetic validation
- Subtotal, tax, and total consistency checks
- Expense categorization
- Confidence scoring
- Duplicate detection using vendor + invoice number
- Human approval through Telegram
- Rejection handling
- Internal invoice ledger
- Google Sheets bookkeeping output
- Pending-invoice lifecycle management
- Separate error-handling workflow
- Error logging through an n8n Data Table

## Architecture

The system is divided into three independent n8n workflows.

```text
                         INVOICE
                            |
                            v
                +-----------------------+
                |  Workflow 1           |
                |  Invoice Processing   |
                +-----------+-----------+
                            |
                  Extract PDF text
                            |
                            v
                       AI extraction
                            |
                            v
                       JSON parsing
                            |
                            v
                    Deterministic
                      validation
                            |
                            v
                    Duplicate check
                            |
                            v
                   pending_invoices
                            |
                            v
                    Telegram approval
                       /          \
                    APPROVE       REJECT
                       |             |
                       v             v
                +---------------------------+
                | Workflow 2                |
                | Approval / Rejection      |
                +-------------+-------------+
                              |
                 +------------+------------+
                 |                         |
              APPROVE                   REJECT
                 |                         |
                 v                         v
          invoice_ledger             rejection handling
                 |
                 v
          Google Sheets
                 |
                 v
       remove pending record


       Any workflow failure
                |
                v
       +-------------------+
       | Workflow 3        |
       | Error Handler     |
       +---------+---------+
                 |
                 v
          invoice_error_log
                 |
                 v
          Telegram alert
```

## Workflow 1: Invoice Processing

Purpose: transform an incoming PDF invoice into a validated record waiting for human approval.

Typical processing stages:

1. Receive/read the invoice PDF.
2. Extract text from the PDF.
3. Send extracted text to the AI model.
4. Parse the AI response as JSON.
5. Validate required fields and arithmetic.
6. Route anomalous records for review.
7. Check for an existing invoice using vendor + invoice number.
8. Save the invoice to `pending_invoices`.
9. Send an approval request to Telegram.

The AI is responsible for interpretation. Business rules and arithmetic are handled deterministically by code. This separation is intentional because a syntactically valid AI response can still contain incorrect numerical values.

The project documentation records validation of:
- required invoice fields
- line-item structure
- quantity × unit price
- line-item subtotal
- subtotal + tax
- confidence
- expense category

## Workflow 2: Approval / Rejection Handling

Purpose: handle the human decision made through Telegram.

The Telegram trigger starts this workflow when the user presses Approve or Reject.

### Approval path

```text
Telegram callback
      |
      v
Find pending invoice
      |
      v
Prepare approved record
      |
      v
Update pending record
      |
      v
Invoice Ledger
      |
      v
Google Sheets
      |
      v
Delete pending record
```

### Rejection path

```text
Telegram callback
      |
      v
Find pending invoice
      |
      v
Handle rejection
      |
      v
Notify user
      |
      v
Delete pending record
```

The pending table therefore acts as temporary workflow state, while the invoice ledger is the durable internal bookkeeping record.

## Workflow 3: Error Handler

Purpose: provide a separate failure-handling path for the other workflows.

The Error Trigger workflow is configured independently from the business workflows. When a configured workflow fails, n8n starts the error workflow with execution and error metadata.

The error handler records useful diagnostic information in `invoice_error_log` and can notify the operator through Telegram.

Typical error information includes:
- workflow name
- execution ID
- failed node
- error message
- error details
- timestamp

## Data Model

The system uses three n8n Data Tables:

| Table | Purpose | Lifecycle |
|---|---|---|
| `pending_invoices` | Temporary invoices waiting for approval | Temporary |
| `invoice_ledger` | Permanent internal bookkeeping/audit record | Persistent |
| `invoice_error_log` | Operational error history | Persistent |

The actual invoice records are runtime data and are intentionally **not stored in GitHub**.

Detailed schemas are documented in [`database/schema.md`](database/schema.md).

## AI Extraction

The project uses an Ollama-backed language model to convert extracted invoice text into structured JSON.

The expected invoice structure includes:

- vendor
- invoice number
- invoice date
- tax ID
- currency
- line items
- subtotal
- tax
- total
- expense category
- confidence

Line items contain:

- description
- quantity
- unit price
- amount

The project uses an allowed expense-category list including Office Supplies, Software, Hardware, Travel, Meals, Utilities, Marketing, Professional Services, Rent, Shipping, and Other.

## Validation Philosophy

The AI is **not trusted blindly**.

For example, if an AI model interprets a line item incorrectly, the resulting JSON can still be perfectly valid JSON. The deterministic validation layer catches arithmetic inconsistencies before the invoice is finalized.

This is especially important for bookkeeping because an AI response can be structurally correct while numerically wrong.

## Duplicate Detection

Invoice identity is based on:

```text
vendor + invoice_number
```

The reason is that invoice numbers are not guaranteed to be globally unique across different vendors.

For example:

```text
ABC Office Supplies / INV-100
XYZ Electronics     / INV-100
```

are not necessarily duplicates.

## Human-in-the-Loop Design

Invoices are not automatically treated as final bookkeeping records after AI extraction.

The workflow follows:

```text
AI extraction
     |
     v
Validation
     |
     v
Duplicate check
     |
     v
Human approval
     |
     +---- Reject
     |
     +---- Approve
```

This prevents questionable AI-extracted records from silently becoming final accounting data.

## Security

This repository contains workflow definitions, documentation, and schemas. It should **not** contain:

- Telegram bot tokens
- Google OAuth secrets
- API keys
- Ollama credentials
- n8n credential databases
- actual invoice PDFs
- customer/vendor private data
- runtime Data Table contents
- local `.env` files

The `.gitignore` file is configured to exclude common secret, database, and runtime-data files.

## Repository Structure

```text
AI-Invoice-Bookkeeping/
|
├── .gitignore
|
├── workflows/
|   ├── Invoice Processing.json
|   ├── Approval_Rejection Handling.json
|   └── Invoice Error Handler.json
|
├── database/
|   └── schema.md
|
└── docs/
    └── architecture.md
```

## Reproducing the Project

A new user who clones this repository receives the workflow blueprints, not the original n8n instance.

To reproduce the system they must:

1. Install n8n.
2. Import the three workflow JSON files.
3. Create the three Data Tables using the documented schemas.
4. Configure their own Telegram credentials.
5. Configure their own Google credentials.
6. Configure their own Ollama credentials/model access.
7. Reconnect resources where required.
8. Configure the Error Workflow relationship.
9. Test the workflows with their own invoice data.

## Limitations

The completed scope intentionally focuses on PDF invoices.

The following are outside the final scope:

- image invoice OCR
- scanned-image OCR
- handwriting recognition
- automatic Gmail ingestion
- production-grade accounting software integration
- multi-user authorization
- enterprise database deployment

These can be treated as future enhancements rather than unfinished core functionality.

## License

No license has been selected for this repository yet. Add a license only when the intended reuse terms are decided.
