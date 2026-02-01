# Purchase Bills (Vendor Invoices) Specification

## Overview
This specification defines the purchase bills system for the accounting platform. Purchase bills represent invoices received from vendors for goods or services purchased, and they form a crucial part of the expense management and accounts payable process.

## Data Model

### Main Bills Table
```sql
CREATE TABLE transactions.bills (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  bill_number VARCHAR(50),
  vendor_id UUID NOT NULL,
  bill_date DATE NOT NULL,
  due_date DATE NOT NULL,
  reference_number VARCHAR(50), -- Vendor's invoice #
  currency CHAR(3) DEFAULT 'USD',
  subtotal DECIMAL(15,2),
  tax_total DECIMAL(15,2),
  total_amount DECIMAL(15,2),
  amount_paid DECIMAL(15,2) DEFAULT 0,
  status ENUM('draft', 'pending', 'approved', 'paid', 'void'),
  approval_status ENUM('pending', 'approved', 'rejected'),
  approved_by UUID,
  approved_at TIMESTAMP,
  journal_entry_id UUID,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,

  FOREIGN KEY (tenant_id) REFERENCES tenants.id,
  FOREIGN KEY (vendor_id) REFERENCES contacts.id,
  FOREIGN KEY (approved_by) REFERENCES users.id,
  FOREIGN KEY (journal_entry_id) REFERENCES transactions.journal_entries.id
);
```

### Bill Lines Table
```sql
CREATE TABLE transactions.bill_lines (
  id UUID PRIMARY KEY,
  bill_id UUID NOT NULL,
  line_number INTEGER,
  description TEXT,
  quantity DECIMAL(10,2),
  unit_price DECIMAL(15,2),
  tax_rate DECIMAL(5,2),
  line_total DECIMAL(15,2),
  account_id UUID, -- Expense account
  created_at TIMESTAMP,

  FOREIGN KEY (bill_id) REFERENCES transactions.bills.id,
  FOREIGN KEY (account_id) REFERENCES chart_of_accounts.id
);
```

## Bill Status Lifecycle

- **Draft**: Bill created but not yet submitted for approval
- **Pending**: Bill submitted for approval
- **Approved**: Bill approved and ready for payment
- **Paid**: Payment processed
- **Void**: Bill cancelled and no longer valid

## Approval Workflow

### Approval Threshold Configuration
- Configurable spending limits per user/role
- Multi-level approval for high-value bills
- Escalation procedures for unresponsive approvers

### Approval Workflow Logic
1. Bill submitted with status 'pending'
2. System determines required approval level based on amount
3. Notification sent to appropriate approver(s)
4. Approvers can approve, reject, or request changes
5. Upon approval, status changes to 'approved'
6. Rejected bills return to submitter with reason

### Multi-Level Approval Process
- Level 1: Up to $X (Department Manager)
- Level 2: $X to $Y (Finance Manager)
- Level 3: Above $Y (CFO/VP Finance)

### Email Notifications
- New bill requires approval notification
- Approval/rejection confirmation
- Reminder notifications for pending approvals
- Mobile push notifications

## Automatic Journal Entry Generation

When a bill is approved (status changed to 'approved'), the system shall automatically generate a journal entry with the following structure:

### Debit Entries:
- Expense Account (Expense Account) - Subtotal amount
- Input Tax Account (Asset/Liability Account) - Tax amount

### Credit Entries:
- Accounts Payable (Liability Account) - Total bill amount

### Example Journal Entry:
For a bill of $1,000 (expense $850, tax $150):

```
Debit:  Expense Account      $850
Debit:  Input Tax (VAT)      $150
Credit: Accounts Payable     $1,000
```

### Journal Automation Logic:
1. Calculate total amounts from bill lines
2. Validate account mappings exist
3. Create journal entry record
4. Link journal entry to bill
5. Post to general ledger
6. Handle errors by marking bill as pending review

## OCR Integration (AI Feature)

### Document Upload
- Support for PDF and image formats (JPG, PNG)
- File size limitations
- Virus scanning for security

### OCR Processing
- Extract vendor name and address
- Extract bill date and due date
- Extract total amount and tax details
- Extract line items (description, quantity, price)
- Extract vendor reference number

### Auto-Population
- Pre-fill bill form with extracted data
- Highlight confidence levels for each field
- Manual verification required for low-confidence extractions

### OCR AI Prompt
```
Extract the following information from the vendor bill:
- Vendor name and address
- Bill date
- Due date
- Reference number
- Total amount (including tax breakdown)
- Line items (description, quantity, unit price, line total)
- Currency
Return the information in a structured JSON format with confidence scores for each field.
```

## 3-Way Matching

### Matching Algorithm
1. Compare bill amount with Purchase Order amount (tolerance: 5%)
2. Compare bill date with Goods Receipt date (tolerance: 30 days)
3. Compare line items across all three documents
4. Flag discrepancies for manual review

### Match Types
- **Exact Match**: All amounts and dates align
- **Tolerance Match**: Minor differences within defined thresholds
- **Exception**: Significant discrepancies requiring review

### Blocking Rules
- Prevent payment if PO not found
- Prevent payment if goods not received
- Require justification for amount variance >5%
- Require approval for date variance >30 days

### 3-Way Match Algorithm
```
IF bill.total_amount within tolerance% of po.total_amount AND
   bill.date within tolerance_days of gr.date AND
   all_line_items_match(bill.lines, po.lines, gr.lines) THEN
   match_status = "Complete"
ELSE IF minor_variance_detected() THEN
   match_status = "Partial"
   flag_for_review()
ELSE
   match_status = "Exception"
   block_payment()
END IF
```

## Payment Scheduling

### Payment Terms Support
- Net 30, Net 60, Net 90
- 2/10 Net 30 (2% discount if paid within 10 days)
- Custom payment terms
- Early payment discount calculations

### Payment Calendar
- Visual calendar showing due dates
- Prioritization based on discount opportunities
- Cash flow impact visualization

### Batch Payment Generation
- Group payments by vendor
- Optimize payment timing for discounts
- Generate payment files for banking systems
- Support for various payment methods (ACH, wire, check)

### Early Payment Discounts
- Calculate potential savings from early payment
- Recommend payment timing to maximize discounts
- Factor in cash flow constraints
- Compare discount rate vs. investment opportunity cost

## Vendor Management Integration

- Link to vendor master data
- Payment history tracking
- Preferred payment methods
- Tax compliance requirements

## Reporting Requirements

The system shall support:
- Accounts payable aging report
- Vendor payment history
- Unpaid bills report
- Approval workflow reports
- OCR processing statistics
- 3-way matching exception reports

## Security & Access Control

- Role-based permissions for bill creation/approval
- Segregation of duties (create vs. approve vs. pay)
- Audit trail for all bill modifications
- Approval authority limits
- Secure document storage
- Compliance with financial regulations

## API Endpoints

### Bill Management
- `POST /api/bills` - Create new bill
- `GET /api/bills` - List bills with filters
- `GET /api/bills/{id}` - Get specific bill
- `PUT /api/bills/{id}` - Update bill
- `DELETE /api/bills/{id}` - Delete draft bill
- `POST /api/bills/{id}/approve` - Approve bill
- `POST /api/bills/{id}/reject` - Reject bill
- `POST /api/bills/upload` - Upload bill for OCR processing

### Bill Lines
- `POST /api/bills/{id}/lines` - Add line item
- `PUT /api/bills/{id}/lines/{lineId}` - Update line item
- `DELETE /api/bills/{id}/lines/{lineId}` - Remove line item

### Matching Operations
- `GET /api/bills/{id}/match` - Check 3-way match status
- `POST /api/bills/{id}/match` - Force match operation

### Payment Operations
- `GET /api/bills/payment-schedule` - Get payment schedule
- `POST /api/bills/generate-payments` - Generate batch payments