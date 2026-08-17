# System Architecture

## 1. Overview

The system is composed of three independent n8n workflows connected through persistent Data Tables and Telegram callbacks.

### Workflow boundaries

```text
Workflow 1
Invoice Processing
    |
    +--> pending_invoices
    |
    +--> Telegram approval request


Workflow 2
Approval / Rejection Handling
    |
    +--> pending_invoices
    +--> invoice_ledger
    +--> Google Sheets


Workflow 3
Invoice Error Handler
    |
    +--> invoice_error_log
    +--> Telegram error notification
```

## 2. Workflow 1: Invoice Processing

```text
PDF input
   |
   v
Read Invoice File
   |
   v
Route / PDF extraction
   |
   v
Basic LLM Chain
   |
   v
Parse AI JSON
   |
   v
Validate Invoice
   |
   v
Anomaly decision
   |
   v
Duplicate check
   |
   v
Prepare approval request
   |
   v
Save Pending Invoice
   |
   v
Send Telegram approval
```

The PDF extraction stage produces text. The AI layer converts that text into structured invoice data. The validation layer then checks business rules independently of the model.

## 3. Workflow 2: Human Decision

Workflow 2 is event-driven. It does not need to be called directly by Workflow 1.

The Telegram message contains Approve and Reject actions. The user's button click produces a Telegram callback event, which starts Workflow 2.

```text
User clicks Approve/Reject
          |
          v
Telegram
          |
          v
Telegram Trigger
          |
          v
Decision handling
```

### Approval

```text
Telegram Trigger
      |
      v
Extract decision + invoice identity
      |
      v
Get pending invoice
      |
      v
Prepare approved record
      |
      v
Update approved invoice
      |
      v
Invoice Ledger
      |
      +--> Google Sheets
      |
      v
Delete pending record
```

### Rejection

```text
Telegram Trigger
      |
      v
Extract decision + invoice identity
      |
      v
Get pending invoice
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

## 4. Workflow 3: Error Handling

The error workflow is intentionally separate from the business workflows.

```text
Workflow 1 or Workflow 2 fails
             |
             v
        Error Trigger
             |
             v
    Prepare error record
             |
             v
      invoice_error_log
             |
             v
       Telegram alert
```

This separation prevents error-handling logic from cluttering the main business workflows.

## 5. Data Lifecycle

```text
                    +------------------+
                    | Incoming invoice |
                    +--------+---------+
                             |
                             v
                    +------------------+
                    | pending_invoices |
                    +--------+---------+
                             |
                       human decision
                        /           \
                       /             \
                  APPROVE           REJECT
                     |                 |
                     v                 v
             +---------------+   delete pending
             | invoice_ledger|
             +-------+-------+
                     |
                     v
              Google Sheets
```

`pending_invoices` is temporary state. `invoice_ledger` is persistent application data. `invoice_error_log` is operational history.

## 6. Design Principles

### AI for interpretation

The model extracts meaning from unstructured invoice text.

### Code for deterministic rules

Code validates arithmetic and strict business rules.

### Human approval for financial finalization

A human decision is required before the record becomes final.

### Persistent state

Data Tables allow the approval workflow to find the invoice after the original processing workflow has ended.

### Separation of concerns

The three workflows have distinct responsibilities:
- processing
- human decision
- failure handling

## 7. Error Handling

The Error Trigger receives information from failed executions, including execution metadata and error information. The error log should preserve enough information to identify:

- what failed
- where it failed
- when it failed
- which execution failed
- what error was reported

This supports debugging without storing the entire n8n execution database in the Git repository.
