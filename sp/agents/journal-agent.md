# AI Journal Entry Agent Specification

## 1. Agent Purpose

The AI Journal Entry Agent is responsible for:
- Suggesting balanced journal entries based on business transactions
- Auto-generating entries from invoices, bills, and payments
- Validating double-entry accounting rules before posting
- Handling multi-currency transactions and conversions

## 2. Journal Generation Logic

### A. From Sales Invoice

**Input:**
- Invoice #INV-001
- Amount: $1,000
- Tax: $150

**Suggested Entry:**
```
Date: Invoice Date
Description: "Sales Invoice #INV-001 - Customer ABC"
Line 1: Debit  - Accounts Receivable   $1,150
Line 2: Credit - Sales Revenue          $1,000
Line 3: Credit - Sales Tax Payable      $150
Status: Pending Approval
```

### B. From Purchase Bill

**Input:**
- Bill from Vendor XYZ
- Amount: $500
- Tax: $75

**Suggested Entry:**
```
Line 1: Debit  - Office Supplies Expense $500
Line 2: Debit  - Input Tax (VAT)         $75
Line 3: Credit - Accounts Payable        $575
```

### C. From Payment

**Input:**
- Payment to Vendor: $500

**Suggested Entry:**
```
Line 1: Debit  - Accounts Payable        $500
Line 2: Credit - Bank Account            $500
```

### D. Multi-Currency Transactions

**Input:**
- Foreign currency invoice: €800
- Exchange rate: 1.1 USD/EUR
- Calculated USD amount: $880

**Suggested Entry:**
```
Date: Transaction Date
Description: "Foreign Invoice EUR 800 (USD 880) - Vendor ABC"
Line 1: Debit  - Inventory              $880
Line 2: Credit - Accounts Payable       $880
Exchange Gain/Loss: $X (if applicable)
```

## 3. Validation Rules (Pre-Post)

### Mandatory Checks
- **Balance Validation**: Total Debits must equal Total Credits
- **Account Existence**: All referenced accounts must exist and be active
- **Header Account Protection**: Prevent posting to header/summary accounts
- **Period Validation**: Ensure accounting period is open for posting
- **Amount Validation**: All amounts must be positive numbers
- **Line Count**: Minimum of 2 lines required for valid entry

### Validation Implementation

```python
def validate_journal_entry(entry):
    """
    Validates a journal entry before allowing posting
    Returns tuple: (is_valid, error_list)
    """
    errors = []

    # Check 1: Balance validation
    total_debits = sum(line.debit for line in entry.lines)
    total_credits = sum(line.credit for line in entry.lines)

    if abs(total_debits - total_credits) > 0.01:  # Allow for rounding differences
        errors.append(f"Debits (${'{:.2f}'.format(total_debits)}) do not equal credits (${'{:.2f}'.format(total_credits)})")

    # Check 2: Account existence and activity
    for line in entry.lines:
        account = get_account_by_code(line.account_code)
        if not account:
            errors.append(f"Account code {line.account_code} does not exist")
        elif not account.is_active:
            errors.append(f"Account code {line.account_code} is inactive")
        elif account.is_header_account:
            errors.append(f"Cannot post to header account {line.account_code}")

    # Check 3: Period validation
    if not is_period_open(entry.date):
        errors.append(f"Accounting period for {entry.date} is closed")

    # Check 4: Amount validation
    for line in entry.lines:
        if line.debit < 0 or line.credit < 0:
            errors.append(f"Negative amounts not allowed: debit={line.debit}, credit={line.credit}")
        if line.debit == 0 and line.credit == 0:
            errors.append(f"Both debit and credit cannot be zero for account {line.account_code}")

    # Check 5: Minimum line count
    if len(entry.lines) < 2:
        errors.append(f"Journal entry must have at least 2 lines, got {len(entry.lines)}")

    return len(errors) == 0, errors
```

### Additional Validation Rules
- **Zero Balance Check**: Verify net effect aligns with business transaction
- **Account Type Validation**: Ensure debits/credits align with account types
- **Duplicate Prevention**: Check for duplicate entries within tolerance
- **Currency Validation**: Ensure proper currency codes and exchange rates

## 4. Recurring Entries

### Template Definitions

#### Monthly Rent Payment
```
Template Name: "Monthly Rent Payment"
Frequency: Monthly
Day: 1st of month
Accounts:
  Debit: Rent Expense (5100)
  Credit: Bank Account (1000)
Amount: Fixed amount per lease agreement
```

#### Monthly Depreciation
```
Template Name: "Monthly Depreciation"
Frequency: Monthly
Day: End of month
Formula: (Asset Cost - Salvage Value) / Useful Life Months
Accounts:
  Debit: Depreciation Expense (5200)
  Credit: Accumulated Depreciation (1500)
```

#### Loan Interest Payment
```
Template Name: "Loan Interest Payment"
Frequency: Monthly
Day: Payment due date
Calculation: Outstanding balance × interest rate × days/365
Accounts:
  Debit: Interest Expense (5300)
  Credit: Bank Account (1000)
```

### Auto-Generation Process
1. **Template Loading**: Load all recurring templates at period start
2. **Date Calculation**: Determine if entry is due this period
3. **Amount Calculation**: Compute amounts based on formulas
4. **Entry Creation**: Generate draft entries for review
5. **Approval Queue**: Submit for required approvals

## 5. Agent Boundaries

### CAN Do
- Suggest balanced journal entries based on transaction data
- Auto-generate entries from invoices, bills, and other documents
- Calculate multi-currency amounts and exchange differences
- Validate accounting rules before allowing posting

### CANNOT Do
- Post entries without proper approval
- Modify or delete already posted journal entries
- Override locked accounting periods

### Requires Approval
- **ALL manual journal entries** regardless of amount
- **Entries exceeding $10,000** threshold
- **Cross-entity transactions** between company subsidiaries
- **Recurring entry generations** before first posting

## 6. Approval Workflow

### Workflow Steps
```
Entry Created (Draft)
    ↓
Submit for Approval
    ↓
Junior Accountant Review (if < $10k)
    ↓
Senior Accountant Review (if > $10k)
    ↓
Posted to GL
    ↓
Immutable Record
```

### Tier Assignment Logic
```python
def determine_approval_tier(entry):
    """
    Determines the approval tier based on entry characteristics
    """
    amount = max(sum(line.debit for line in entry.lines),
                 sum(line.credit for line in entry.lines))

    # Check for special circumstances requiring higher approval
    has_special_accounts = any(is_special_approval_account(line.account_code)
                              for line in entry.lines)

    is_cross_entity = entry.metadata.get('cross_entity', False)
    is_manual_entry = entry.metadata.get('entry_type') == 'manual'

    if amount > 10000 or has_special_accounts or is_cross_entity or is_manual_entry:
        return 'senior_accountant'
    else:
        return 'junior_accountant'

def is_special_approval_account(account_code):
    """
    Accounts that always require senior approval
    """
    special_accounts = [
        '8000',  # Owner's Equity
        '2000',  # Loans Payable
        '1000',  # Cash/Bank (high amounts)
        '5000'   # All expense accounts (for large amounts)
    ]
    return account_code in special_accounts
```

### Approval Process States
- **DRAFT**: Entry created but not submitted
- **PENDING_APPROVAL**: Submitted for review
- **APPROVED**: Approved and ready to post
- **REJECTED**: Returned with comments
- **POSTED**: Successfully entered into GL
- **CANCELLED**: Voided after posting (via reversing entry)

## 7. API Integration

### Endpoint: POST /api/v1/ai/suggest-journal

#### Request Body
```json
{
  "source_type": "invoice",
  "source_id": "inv_123",
  "invoice_data": {
    "total": 1000,
    "tax": 150,
    "customer_id": "cust_456",
    "currency": "USD",
    "exchange_rate": 1.0
  },
  "metadata": {
    "created_by": "user_789",
    "business_unit": "sales_dept",
    "project_code": "proj_101"
  }
}
```

#### Response Schema
```json
{
  "suggested_entry": {
    "entry_id": "je_temp_12345",
    "description": "Sales Invoice INV-001 - Customer ABC",
    "date": "2024-01-15",
    "reference_number": "INV-001",
    "lines": [
      {
        "account_code": "1200",
        "account_name": "Accounts Receivable",
        "debit": 1150,
        "credit": 0,
        "currency": "USD",
        "foreign_amount": 1150
      },
      {
        "account_code": "4000",
        "account_name": "Sales Revenue",
        "debit": 0,
        "credit": 1000,
        "currency": "USD",
        "foreign_amount": 1000
      },
      {
        "account_code": "2100",
        "account_name": "Sales Tax Payable",
        "debit": 0,
        "credit": 150,
        "currency": "USD",
        "foreign_amount": 150
      }
    ],
    "validation": {
      "is_balanced": true,
      "errors": [],
      "warnings": []
    },
    "metadata": {
      "generated_by": "ai_journal_agent",
      "confidence": 0.98
    }
  },
  "requires_approval": true,
  "approval_tier": "junior_accountant",
  "next_steps": [
    "Submit for junior accountant review",
    "Verify customer account details",
    "Confirm tax rate application"
  ]
}
```

#### Alternative Request Types

**Purchase Bill Request:**
```json
{
  "source_type": "purchase_bill",
  "source_id": "bill_456",
  "bill_data": {
    "vendor_id": "vend_789",
    "subtotal": 500,
    "tax": 75,
    "total": 575
  }
}
```

**Payment Request:**
```json
{
  "source_type": "payment",
  "source_id": "pay_101",
  "payment_data": {
    "vendor_id": "vend_789",
    "amount": 500,
    "payment_method": "check"
  }
}
```

## 8. Entry Generation Algorithms

### Sales Invoice Processing
```python
def generate_sales_invoice_entry(invoice_data):
    """
    Generates journal entry for sales invoice
    """
    lines = []

    # Calculate total receivable
    total_receivable = invoice_data.total + invoice_data.tax

    # Line 1: Debit Accounts Receivable
    lines.append({
        'account_code': '1200',  # Accounts Receivable
        'debit': total_receivable,
        'credit': 0
    })

    # Line 2: Credit Sales Revenue
    lines.append({
        'account_code': '4000',  # Sales Revenue
        'debit': 0,
        'credit': invoice_data.total
    })

    # Line 3: Credit Sales Tax Payable
    lines.append({
        'account_code': '2100',  # Sales Tax Payable
        'debit': 0,
        'credit': invoice_data.tax
    })

    return {
        'description': f'Sales Invoice {invoice_data.invoice_number} - {invoice_data.customer_name}',
        'lines': lines
    }
```

### Purchase Bill Processing
```python
def generate_purchase_bill_entry(bill_data):
    """
    Generates journal entry for purchase bill
    """
    lines = []

    # Determine appropriate expense account based on vendor/category
    expense_account = determine_expense_account(bill_data.vendor_id, bill_data.category)

    # Line 1: Debit Expense Account
    lines.append({
        'account_code': expense_account,
        'debit': bill_data.subtotal,
        'credit': 0
    })

    # Line 2: Debit Tax Account (if applicable)
    if bill_data.tax > 0:
        lines.append({
            'account_code': '1300',  # Input Tax/VAT
            'debit': bill_data.tax,
            'credit': 0
        })

    # Line 3: Credit Accounts Payable
    lines.append({
        'account_code': '2000',  # Accounts Payable
        'debit': 0,
        'credit': bill_data.total
    })

    return {
        'description': f'Purchase Bill {bill_data.bill_number} - {bill_data.vendor_name}',
        'lines': lines
    }
```

### Multi-Currency Processing
```python
def calculate_currency_adjustment(original_amount, exchange_rate, functional_currency):
    """
    Calculates foreign currency adjustment and potential gain/loss
    """
    converted_amount = original_amount * exchange_rate

    # Check if original transaction was recorded at different rate
    # Calculate gain/loss if rates differ
    if original_exchange_rate and original_exchange_rate != exchange_rate:
        gain_loss = (original_amount * exchange_rate) - (original_amount * original_exchange_rate)

        gain_loss_account = '7000' if gain_loss > 0 else '7100'  # Foreign Exchange Gain/Loss

        return {
            'converted_amount': converted_amount,
            'gain_loss': abs(gain_loss),
            'gain_loss_account': gain_loss_account
        }

    return {'converted_amount': converted_amount}
```

## 9. Recurring Entry Templates

### Template Structure
```json
{
  "template_id": "rent_monthly_001",
  "name": "Monthly Rent Payment",
  "frequency": "monthly",
  "day_of_month": 1,
  "accounts": [
    {
      "account_code": "5100",
      "account_name": "Rent Expense",
      "direction": "debit",
      "amount_type": "fixed",
      "amount_value": 2500.00
    },
    {
      "account_code": "1000",
      "account_name": "Bank Account",
      "direction": "credit",
      "amount_type": "calculated",
      "formula": "sum(debit_lines)"
    }
  ],
  "active": true,
  "next_due_date": "2024-02-01",
  "auto_generate": true,
  "requires_approval": true
}
```

### Template Management
- **Creation**: Admin creates templates with account mapping
- **Activation**: Enable/disable templates as needed
- **Modification**: Update amounts, dates, or accounts
- **Monitoring**: Track generation and approval status

## 10. Security and Audit Controls

### Access Control
- **Read Access**: View journal entries and suggestions
- **Create Access**: Generate new journal entries
- **Approve Access**: Approve entries within authority limits
- **Post Access**: Final posting to general ledger

### Audit Trail
- **Creation Log**: Who created the entry and when
- **Modification Log**: All changes with timestamps
- **Approval Log**: Complete approval chain
- **Posting Log**: Final posting with timestamp
- **Reversal Log**: Any reversals with reason

### Data Integrity
- **Immutable Posted Entries**: Once posted, entries cannot be modified
- **Balancing Validation**: Mathematical accuracy verification
- **Account Locking**: Prevent posting to restricted accounts
- **Period Controls**: Enforce accounting period boundaries