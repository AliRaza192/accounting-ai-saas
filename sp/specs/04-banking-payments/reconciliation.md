# Bank Reconciliation Specification

## Overview
This specification defines the bank reconciliation system for the accounting platform. Bank reconciliation ensures that the company's book balance matches the bank statement balance by identifying and explaining differences between the two.

## Data Model

### Reconciliations Table
```sql
CREATE TABLE banking.reconciliations (
  id UUID PRIMARY KEY,
  bank_account_id UUID NOT NULL,
  statement_date DATE NOT NULL,
  statement_balance DECIMAL(15,2),
  gl_balance DECIMAL(15,2),
  status ENUM('in_progress', 'balanced', 'unbalanced'),
  difference DECIMAL(15,2),
  reconciled_by UUID,
  reconciled_at TIMESTAMP,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,

  FOREIGN KEY (bank_account_id) REFERENCES banking.bank_accounts.id,
  FOREIGN KEY (reconciled_by) REFERENCES users.id
);
```

### Reconciliation Items Table
```sql
CREATE TABLE banking.reconciliation_items (
  id UUID PRIMARY KEY,
  reconciliation_id UUID NOT NULL,
  bank_transaction_id UUID,
  journal_entry_id UUID,
  item_type ENUM('bank_only', 'gl_only', 'matched'),
  amount DECIMAL(15,2),
  is_matched BOOLEAN DEFAULT FALSE,
  matched_at TIMESTAMP,
  matched_by UUID,

  FOREIGN KEY (reconciliation_id) REFERENCES banking.reconciliations.id,
  FOREIGN KEY (bank_transaction_id) REFERENCES banking.bank_transactions.id,
  FOREIGN KEY (journal_entry_id) REFERENCES transactions.journal_entries.id,
  FOREIGN KEY (matched_by) REFERENCES users.id
);
```

## Matching Process

### Auto-Matching Rules
1. **Exact Match**: Same amount and date (±3 days tolerance)
2. **Fuzzy Match**: Similar description patterns and amounts
3. **Rule-Based Match**: Based on bank rules and patterns
4. **AI-Assisted Match**: Machine learning for complex patterns

### Matching Algorithm (AI-Assisted)
```
FOR each unreconciled bank transaction DO
  1. Check for exact matches (amount ± tolerance and date ± 3 days)
  2. Apply fuzzy matching on description using similarity algorithms
  3. Use bank rules for pattern-based matching
  4. Apply AI model trained on historical matches
  5. Calculate confidence score for each potential match
  6. Auto-match if confidence > 90%, suggest if 70-90%, manual if < 70%
END FOR

FOR each unreconciled journal entry DO
  1. Check for matching bank transactions
  2. Apply same matching logic as above
END FOR
```

### Many-to-One Matching
- Support for split transactions (one bank transaction matches multiple journal entries)
- Consolidated deposits/withdrawals
- Fee batches and interest consolidations

## Unreconciled Items Classification

### Outstanding Checks
- Checks issued but not yet cleared by bank
- Negative amounts in GL but not in bank statement
- Time-sensitive items requiring attention

### Deposits in Transit
- Deposits made but not yet reflected in bank statement
- Positive amounts in GL but not in bank statement
- Usually cleared in subsequent period

### Bank Fees Not Recorded
- Bank charges not yet entered in GL
- Negative amounts in bank but not in GL
- Requires adjustment entries

### Interest Earned
- Interest credited by bank not recorded in GL
- Positive amounts in bank but not in GL
- Requires adjustment entries

## Reconciliation Workflow

### Step 1: Preparation
1. Obtain bank statement for the period
2. Ensure all transactions are imported
3. Verify GL balance as of statement date
4. Prepare list of unreconciled items

### Step 2: Matching Process
1. Run automatic matching algorithm
2. Review and confirm suggested matches
3. Manually match remaining items
4. Investigate and resolve discrepancies

### Step 3: Analysis
1. Identify outstanding deposits and checks
2. Record bank fees and interest not in GL
3. Prepare adjusting entries if needed
4. Calculate reconciliation difference

### Step 4: Completion
1. Verify reconciliation balances
2. Document any adjustments made
3. Save reconciliation record
4. Mark as complete with user signature

## Reconciliation Report

### Standard Reconciliation Format
```
Account: [Account Name]                    Statement Date: [Date]
Statement Balance:                        $[XX,XXX.XX]

Book Balance:                            $[XX,XXX.XX]
Add: Deposits in Transit                  $[XXX.XX]
Less: Outstanding Checks                 ($[XXX.XX])
Adjustments:                             $[±XXX.XX]
= Reconciled Balance:                     $[XX,XXX.XX]

Bank Statement Balance:                   $[XX,XXX.XX] ✓
Difference:                               $[0.00]
Status: Balanced

Reconciled by: [Name]                    Date: [Date]
```

### Variance Report (when unbalanced)
```
Account: [Account Name]                    Statement Date: [Date]
Statement Balance:                        $[XX,XXX.XX]

Book Balance:                            $[XX,XXX.XX]
Add: Deposits in Transit                  $[XXX.XX]
Less: Outstanding Checks                 ($[XXX.XX])
Adjustments:                             $[±XXX.XX]
= Reconciled Balance:                     $[XX,XXX.XX]

Bank Statement Balance:                   $[XX,XXX.XX]
Difference:                              ($[±X.XX]) ← ALERT!

Unreconciled Items:
- Bank Only Items: [Count] items totaling $[XXX.XX]
- GL Only Items: [Count] items totaling $[XXX.XX]

Status: Unbalanced - Review Required
```

## Period Close Requirements

### Pre-Close Validation
- All bank accounts must be reconciled to the period end date
- No outstanding unreconciled items older than 30 days
- Zero balance difference in all reconciliations

### Close Process Impact
- Prevent period closure if any bank account is unreconciled
- Generate warnings for accounts not reconciled within 30 days
- Flag accounts with significant unreconciled differences

### Warning System
- Alert when reconciliation is >30 days overdue
- Escalating notifications for >60 and >90 days
- Management dashboard for oversight

## Report Templates

### Monthly Reconciliation Summary
- Account balances and reconciliation status
- Age of last reconciliation
- Total unreconciled amounts
- Exception items requiring attention

### Outstanding Items Report
- Detailed listing of unreconciled transactions
- Aging analysis (current, 30+, 60+, 90+ days)
- Vendor/customer breakdown for outstanding items

### Reconciliation Activity Report
- History of reconciliation activities
- User activity and responsibilities
- Timing and efficiency metrics

## Security & Access Control

- Role-based access to reconciliation features
- Segregation of duties (prepare vs. approve reconciliations)
- Audit trail for all reconciliation activities
- Digital signatures for completed reconciliations
- Access logs for compliance requirements

## API Endpoints

### Reconciliation Management
- `POST /api/banking/reconciliations` - Start new reconciliation
- `GET /api/banking/reconciliations` - List reconciliations
- `GET /api/banking/reconciliations/{id}` - Get reconciliation details
- `PUT /api/banking/reconciliations/{id}` - Update reconciliation
- `POST /api/banking/reconciliations/{id}/complete` - Complete reconciliation
- `POST /api/banking/reconciliations/{id}/match` - Match items in reconciliation

### Item Management
- `POST /api/banking/reconciliations/{id}/items/match` - Match bank/Gl items
- `DELETE /api/banking/reconciliations/{id}/items/unmatch` - Unmatch items
- `GET /api/banking/reconciliations/{id}/items/outstanding` - Get outstanding items

### Reports
- `GET /api/banking/reports/reconciliation-summary` - Monthly summary
- `GET /api/banking/reports/outstanding-items` - Outstanding items report
- `GET /api/banking/reports/reconciliation-activity` - Activity report