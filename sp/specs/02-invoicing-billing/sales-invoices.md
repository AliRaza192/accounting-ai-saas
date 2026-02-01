# Sales Invoicing Specification

## Overview
This specification defines the sales invoicing system for the accounting platform. Sales invoices represent bills sent to customers for goods or services provided, and they form a crucial part of the revenue recognition process.

## Data Model

### Main Invoice Table
```sql
CREATE TABLE transactions.invoices (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  invoice_number VARCHAR(50) NOT NULL,
  customer_id UUID NOT NULL,
  invoice_date DATE NOT NULL,
  due_date DATE NOT NULL,
  currency CHAR(3) DEFAULT 'USD',
  subtotal DECIMAL(15,2) NOT NULL,
  tax_total DECIMAL(15,2) DEFAULT 0,
  discount_total DECIMAL(15,2) DEFAULT 0,
  total_amount DECIMAL(15,2) NOT NULL,
  amount_paid DECIMAL(15,2) DEFAULT 0,
  status ENUM('draft', 'sent', 'partial', 'paid', 'overdue', 'void'),
  terms TEXT,
  notes TEXT,
  journal_entry_id UUID,
  created_at TIMESTAMP,

  UNIQUE(tenant_id, invoice_number),
  FOREIGN KEY (tenant_id) REFERENCES tenants.id,
  FOREIGN KEY (customer_id) REFERENCES contacts.id,
  FOREIGN KEY (journal_entry_id) REFERENCES transactions.journal_entries.id
);
```

### Invoice Lines Table
```sql
CREATE TABLE transactions.invoice_lines (
  id UUID PRIMARY KEY,
  invoice_id UUID NOT NULL,
  line_number INTEGER,
  description TEXT,
  quantity DECIMAL(10,2),
  unit_price DECIMAL(15,2),
  tax_rate DECIMAL(5,2),
  discount_percent DECIMAL(5,2),
  line_total DECIMAL(15,2),
  account_id UUID, -- Revenue account
  created_at TIMESTAMP,

  FOREIGN KEY (invoice_id) REFERENCES transactions.invoices.id,
  FOREIGN KEY (account_id) REFERENCES chart_of_accounts.id
);
```

## Invoice Status Lifecycle

- **Draft**: Invoice created but not yet sent to customer
- **Sent**: Invoice sent to customer but payment not yet received
- **Partial**: Partial payment received
- **Paid**: Full payment received
- **Overdue**: Due date passed without full payment
- **Void**: Invoice cancelled and no longer valid

## Business Rules

1. Invoice number must be unique per tenant
2. Invoice date cannot be in the future
3. Due date must be equal to or after invoice date
4. Total amount = subtotal + tax_total - discount_total
5. Amount paid cannot exceed total amount
6. Tax calculations must follow configured tax rates
7. Currency conversion should be applied for multi-currency transactions

## Automatic Journal Entry Generation

When an invoice is posted (status changed from 'draft' to 'sent'), the system shall automatically generate a journal entry with the following structure:

### Debit Entries:
- Accounts Receivable (Asset Account) - Total invoice amount

### Credit Entries:
- Revenue Account (Revenue Account) - Subtotal amount
- Tax Liability Account (Liability Account) - Tax amount
- Discount Account (Contra-revenue Account) - Discount amount (if applicable)

### Example Journal Entry:
For an invoice of $1,000 (revenue $850, tax $150):

```
Debit:  Accounts Receivable  $1,000
Credit: Sales Revenue        $850
Credit: Sales Tax Payable    $150
```

### Journal Automation Logic:
1. Calculate total amounts from invoice lines
2. Validate account mappings exist
3. Create journal entry record
4. Link journal entry to invoice
5. Post to general ledger
6. Handle errors by marking invoice as pending review

## Payment Processing Integration

When payments are received against invoices:
1. Update amount_paid field
2. Adjust invoice status based on payment amount
3. Generate corresponding cash receipt journal entry
4. Optionally create payment allocation records

## Reporting Requirements

The system shall support:
- Invoice aging reports
- Revenue recognition reports
- Customer outstanding balances
- Invoice summary reports by period
- Unpaid invoices report

## Invoice Numbering

- Sequential auto-increment per tenant
- Configurable prefix (e.g., INV-2024-)
- No gaps allowed in numbering sequence
- Voided invoices retain their number in the sequence
- Format: {prefix}{sequential_number} (e.g., INV-2024-0001)

## Payment Tracking

- Support for partial payments against invoices
- Overpayment handling with credit balance creation
- Aging calculation (30, 60, 90+ days overdue)
- Payment application logic to match payments to invoices
- Outstanding balance tracking

## Recurring Invoices

- Template-based recurring invoice creation
- Frequency options: daily, weekly, monthly, yearly
- Auto-send option for scheduled delivery
- Configurable end date or maximum occurrence count
- Pause/resume functionality
- Modification tracking for recurring templates

## PDF Generation

- Professional invoice template with customizable branding
- Company logo and contact details inclusion
- Clear line items table with quantities and pricing
- Payment instructions and due date highlighting
- Multi-language support
- Print-optimized layout
- Download and email-ready format

## Email Delivery

- Automated sending to customer email addresses
- CC/BCC support for additional stakeholders
- Delivery status tracking
- Payment link inclusion for online payments
- Follow-up reminder scheduling
- Email template customization

## AI Features

- Intelligent auto-fill from previous invoices
- Product and service suggestion based on customer history
- Dynamic pricing recommendations
- Predictive payment date estimation
- Anomaly detection for unusual invoice patterns

## CRUD Operations

### Create Invoice
- Validate required fields
- Auto-generate invoice number
- Calculate totals and taxes
- Save as draft initially
- Create associated line items

### Read Invoice
- Retrieve complete invoice details
- Include customer information
- Show line item breakdown
- Display payment history
- Provide status updates

### Update Invoice
- Allow modifications only in 'draft' status
- Maintain audit trail of changes
- Recalculate totals when items change
- Prevent changes after payment processing
- Update related journal entries if needed

### Delete Invoice
- Only allowed for 'draft' status invoices
- Remove related line items
- Delete associated journal entries if unposted
- Maintain soft-delete for audit purposes

## API Endpoints

### Invoice Management
- `POST /api/invoices` - Create new invoice
- `GET /api/invoices` - List invoices with filters
- `GET /api/invoices/{id}` - Get specific invoice
- `PUT /api/invoices/{id}` - Update invoice
- `DELETE /api/invoices/{id}` - Delete draft invoice
- `POST /api/invoices/{id}/send` - Send invoice to customer
- `POST /api/invoices/{id}/void` - Void invoice

### Invoice Lines
- `POST /api/invoices/{id}/lines` - Add line item
- `PUT /api/invoices/{id}/lines/{lineId}` - Update line item
- `DELETE /api/invoices/{id}/lines/{lineId}` - Remove line item

### Payment Operations
- `POST /api/invoices/{id}/payments` - Apply payment
- `GET /api/invoices/{id}/payments` - Get payment history
- `GET /api/invoices/aging` - Get aging report

### Recurring Invoices
- `POST /api/recurring-invoices` - Create recurring template
- `GET /api/recurring-invoices/{id}/generate` - Generate from template

## PDF Template Specification

### Header Section
- Company logo (top left)
- Company name and address
- Invoice title and number
- Date information (invoice date, due date)

### Customer Information
- Bill to: customer name and address
- Shipping to: if different
- Customer contact information

### Line Items Table
- Item/Description column
- Quantity column
- Unit price column
- Line total column
- Clear subtotal calculation

### Totals Section
- Subtotal amount
- Tax breakdown
- Discount information
- Total amount due
- Amount paid and balance due

### Footer Section
- Terms and conditions
- Payment instructions
- Bank details for payment
- Thank you message

## Email Template

### Subject Line
"[Company Name] Invoice #{invoice_number} for {amount}"

### Body Content
- Personalized greeting to customer
- Invoice summary with number and amount
- Due date emphasis
- Payment link/button
- Download attachment option
- Contact information for questions
- Professional closing

## Security & Access Control

- Only authorized users can create/modify invoices
- Invoice numbers must follow sequential numbering
- Audit trail for all invoice modifications
- Tenant isolation for multi-tenant environments
- Role-based permissions for invoice operations
- Secure PDF generation and delivery