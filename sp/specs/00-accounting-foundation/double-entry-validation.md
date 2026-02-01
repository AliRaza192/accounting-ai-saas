# DOUBLE-ENTRY VALIDATION SPECIFICATION - AccountingAI Pro

**Specification Version:** 1.0
**Effective Date:** January 31, 2026
**Classification:** Foundation Specification

---

## OVERVIEW

This specification defines the validation rules and system behavior for double-entry bookkeeping in the AccountingAI Pro system. All transactions must comply with double-entry principles where every debit has a corresponding credit and vice versa.

---

## 1. TRANSACTION STRUCTURE

### 1.1 Transaction Object
```typescript
interface Transaction {
  id: UUID
  date: Date
  description: String
  lines: JournalLine[]  // minimum 2 lines
  total_debit: Decimal(precision=15, scale=2)
  total_credit: Decimal(precision=15, scale=2)
  status: enum('draft', 'posted', 'voided')
  created_at: DateTime
  updated_at: DateTime
  created_by: UUID
}
```

### 1.2 JournalLine Object
```typescript
interface JournalLine {
  account_id: UUID  // Foreign Key to ChartOfAccounts
  debit: Decimal >= 0
  credit: Decimal >= 0
  description: String
  constraint: (debit = 0 OR credit = 0) AND NOT (debit = 0 AND credit = 0)
  created_at: DateTime
  updated_at: DateTime
}
```

### 1.3 Constraints Explanation
- Each line must have either a debit OR credit amount, but not both
- Each line must have at least one non-zero amount
- Transaction must have at least 2 lines to ensure double-entry

---

## 2. VALIDATION RULES (Execute in Order)

### 2.1 Rule 1: Minimum Lines Validation
**Requirement:** Every transaction MUST have >= 2 lines

**Implementation:**
```typescript
function validateMinimumLines(transaction: Transaction): ValidationResult {
  if (transaction.lines.length < 2) {
    return {
      valid: false,
      error: "Transaction must have at least one debit and one credit"
    };
  }
  return { valid: true };
}
```

**Error Message:** "Transaction must have at least one debit and one credit"

### 2.2 Rule 2: Balance Check Validation
**Requirement:** SUM(debit) MUST = SUM(credit) with 2 decimal precision

**Implementation:**
```typescript
function validateBalance(transaction: Transaction): ValidationResult {
  const totalDebit = transaction.lines.reduce((sum, line) =>
    sum.add(line.debit), new Decimal(0));

  const totalCredit = transaction.lines.reduce((sum, line) =>
    sum.add(line.credit), new Decimal(0));

  const difference = totalDebit.minus(totalCredit);

  // Check if difference is within acceptable rounding tolerance (0.01)
  if (!difference.abs().lessThanOrEqualTo(new Decimal(0.01))) {
    return {
      valid: false,
      error: `Transaction out of balance by ${difference.toFixed(2)}`
    };
  }

  return { valid: true };
}
```

**Error Message:** "Transaction out of balance by ${difference}"

### 2.3 Rule 3: Zero Amount Check Validation
**Requirement:** No line can have both debit=0 AND credit=0

**Implementation:**
```typescript
function validateZeroAmounts(transaction: Transaction): ValidationResult {
  for (let i = 0; i < transaction.lines.length; i++) {
    const line = transaction.lines[i];
    if (line.debit.isZero() && line.credit.isZero()) {
      return {
        valid: false,
        error: `Line ${i + 1} has no amount`
      };
    }
  }
  return { valid: true };
}
```

**Error Message:** "Line {line_number} has no amount"

### 2.4 Rule 4: Dual Amount Check Validation
**Requirement:** No line can have both debit>0 AND credit>0

**Implementation:**
```typescript
function validateDualAmounts(transaction: Transaction): ValidationResult {
  for (let i = 0; i < transaction.lines.length; i++) {
    const line = transaction.lines[i];
    if (!line.debit.isZero() && !line.credit.isZero()) {
      return {
        valid: false,
        error: `Line ${i + 1} cannot have both debit and credit`
      };
    }
  }
  return { valid: true };
}
```

**Error Message:** "Line {line_number} cannot have both debit and credit"

### 2.5 Rule 5: Account Validity Check
**Requirement:** All accounts must exist and be active; no posting to header accounts

**Implementation:**
```typescript
async function validateAccountValidity(transaction: Transaction): Promise<ValidationResult> {
  for (const line of transaction.lines) {
    const account = await getAccountById(line.account_id);

    if (!account || account.status !== 'active') {
      return {
        valid: false,
        error: `Account ${account?.code || line.account_id} is invalid or inactive`
      };
    }

    if (account.account_type === 'HEADER') {
      return {
        valid: false,
        error: `Cannot post to header account ${account.code}`
      };
    }
  }
  return { valid: true };
}
```

**Error Message:** "Account {account_code} is invalid or inactive"

### 2.6 Rule 6: Period Lock Check
**Requirement:** Transaction date must be in open period

**Implementation:**
```typescript
async function validatePeriodLock(transaction: Transaction): Promise<ValidationResult> {
  const period = await getAccountingPeriod(transaction.date);

  if (!period || period.status === 'closed') {
    return {
      valid: false,
      error: `Cannot post to closed period ${period?.period_name || transaction.date}`
    };
  }

  return { valid: true };
}
```

**Error Message:** "Cannot post to closed period {period}"

---

## 3. VALIDATION EXECUTION ORDER

### 3.1 Sequential Validation Pipeline
```typescript
async function validateTransaction(transaction: Transaction): Promise<ValidationResult> {
  // Rule 1: Minimum Lines
  const minLinesResult = validateMinimumLines(transaction);
  if (!minLinesResult.valid) return minLinesResult;

  // Rule 2: Balance Check
  const balanceResult = validateBalance(transaction);
  if (!balanceResult.valid) return balanceResult;

  // Rule 3: Zero Amount Check
  const zeroAmountResult = validateZeroAmounts(transaction);
  if (!zeroAmountResult.valid) return zeroAmountResult;

  // Rule 4: Dual Amount Check
  const dualAmountResult = validateDualAmounts(transaction);
  if (!dualAmountResult.valid) return dualAmountResult;

  // Rule 5: Account Validity (async)
  const accountValidResult = await validateAccountValidity(transaction);
  if (!accountValidResult.valid) return accountValidResult;

  // Rule 6: Period Lock (async)
  const periodLockResult = await validatePeriodLock(transaction);
  if (!periodLockResult.valid) return periodLockResult;

  // All validations passed
  return { valid: true };
}
```

---

## 4. PRE-COMMIT HOOKS

### 4.1 Database Transaction Hooks
```sql
-- PostgreSQL trigger example
CREATE OR REPLACE FUNCTION validate_double_entry()
RETURNS TRIGGER AS $$
DECLARE
    total_debit DECIMAL(15,2);
    total_credit DECIMAL(15,2);
    line_count INTEGER;
BEGIN
    -- Count lines for this transaction
    SELECT COUNT(*) INTO line_count
    FROM journal_lines
    WHERE transaction_id = NEW.id;

    IF line_count < 2 THEN
        RAISE EXCEPTION 'Transaction must have at least one debit and one credit';
    END IF;

    -- Calculate totals
    SELECT
        COALESCE(SUM(debit), 0),
        COALESCE(SUM(credit), 0)
    INTO total_debit, total_credit
    FROM journal_lines
    WHERE transaction_id = NEW.id;

    -- Check balance
    IF ABS(total_debit - total_credit) > 0.01 THEN
        RAISE EXCEPTION 'Transaction out of balance by %', (total_debit - total_credit);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER double_entry_validation_trigger
    BEFORE INSERT OR UPDATE ON transactions
    FOR EACH ROW
    EXECUTE FUNCTION validate_double_entry();
```

### 4.2 Application-Level Pre-commit Hook
```typescript
async function preCommitValidation(transaction: Transaction): Promise<void> {
  const validationResult = await validateTransaction(transaction);

  if (!validationResult.valid) {
    throw new ValidationError(validationResult.error);
  }

  // If validation passes, proceed with database transaction
  await db.transaction(async (trx) => {
    // Insert transaction and lines
    await trx.insertInto('transactions').values(transaction).execute();
    await trx.insertInto('journal_lines').values(transaction.lines).execute();
  });
}
```

---

## 5. POST-COMMIT VERIFICATION

### 5.1 Scheduled Verification Job
```typescript
async function runDailyVerification(): Promise<VerificationReport> {
  const report: VerificationReport = {
    total_transactions: 0,
    unbalanced_transactions: [],
    anomalies_found: false,
    general_ledger_balance: 0
  };

  // Find all posted transactions
  const transactions = await db.selectFrom('transactions')
    .where('status', '=', 'posted')
    .selectAll()
    .execute();

  report.total_transactions = transactions.length;

  for (const transaction of transactions) {
    const lines = await db.selectFrom('journal_lines')
      .where('transaction_id', '=', transaction.id)
      .selectAll()
      .execute();

    const totalDebit = lines.reduce((sum, line) => sum.add(line.debit), new Decimal(0));
    const totalCredit = lines.reduce((sum, line) => sum.add(line.credit), new Decimal(0));
    const difference = totalDebit.minus(totalCredit);

    if (difference.abs().greaterThan(new Decimal(0.01))) {
      report.unbalanced_transactions.push({
        transaction_id: transaction.id,
        difference: difference.toString(),
        date: transaction.date
      });
      report.anomalies_found = true;
    }
  }

  // Verify general ledger balance
  const generalLedgerBalance = await calculateGeneralLedgerBalance();
  report.general_ledger_balance = generalLedgerBalance;

  if (!generalLedgerBalance.isZero()) {
    report.anomalies_found = true;
  }

  return report;
}
```

### 5.2 Alert System
```typescript
function handleVerificationAnomalies(report: VerificationReport): void {
  if (report.anomalies_found) {
    // Send alerts to finance team
    sendAlert({
      type: 'DOUBLE_ENTRY_ANOMALY',
      message: `Daily verification found ${report.unbalanced_transactions.length} unbalanced transactions`,
      data: report
    });

    // Log to audit trail
    logAuditEvent({
      action: 'VERIFICATION_ANOMALY_DETECTED',
      details: report,
      severity: 'HIGH'
    });
  }
}
```

---

## 6. TESTING REQUIREMENTS

### 6.1 Unit Tests for Each Validation Rule

```typescript
describe('Double-Entry Validation', () => {
  describe('Rule 1: Minimum Lines', () => {
    test('should reject transaction with only 1 line', () => {
      const transaction = {
        id: 'uuid',
        date: '2026-01-31',
        description: 'Test',
        lines: [{ account_id: 'acc1', debit: 100, credit: 0, description: 'Test' }],
        total_debit: 100,
        total_credit: 0,
        status: 'draft'
      };

      const result = validateMinimumLines(transaction);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('at least one debit and one credit');
    });

    test('should accept transaction with 2 or more lines', () => {
      const transaction = {
        id: 'uuid',
        date: '2026-01-31',
        description: 'Test',
        lines: [
          { account_id: 'acc1', debit: 100, credit: 0, description: 'Test' },
          { account_id: 'acc2', debit: 0, credit: 100, description: 'Test' }
        ],
        total_debit: 100,
        total_credit: 100,
        status: 'draft'
      };

      const result = validateMinimumLines(transaction);
      expect(result.valid).toBe(true);
    });
  });

  describe('Rule 2: Balance Check', () => {
    test('should reject unbalanced transaction', () => {
      const transaction = {
        id: 'uuid',
        date: '2026-01-31',
        description: 'Test',
        lines: [
          { account_id: 'acc1', debit: 100, credit: 0, description: 'Test' },
          { account_id: 'acc2', debit: 0, credit: 99, description: 'Test' }  // Unbalanced
        ],
        total_debit: 100,
        total_credit: 99,
        status: 'draft'
      };

      const result = validateBalance(transaction);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('out of balance');
    });

    test('should accept balanced transaction', () => {
      const transaction = {
        id: 'uuid',
        date: '2026-01-31',
        description: 'Test',
        lines: [
          { account_id: 'acc1', debit: 100, credit: 0, description: 'Test' },
          { account_id: 'acc2', debit: 0, credit: 100, description: 'Test' }
        ],
        total_debit: 100,
        total_credit: 100,
        status: 'draft'
      };

      const result = validateBalance(transaction);
      expect(result.valid).toBe(true);
    });
  });

  describe('Rule 3: Zero Amount Check', () => {
    test('should reject line with zero debit and zero credit', () => {
      const transaction = {
        id: 'uuid',
        date: '2026-01-31',
        description: 'Test',
        lines: [
          { account_id: 'acc1', debit: 100, credit: 0, description: 'Test' },
          { account_id: 'acc2', debit: 0, credit: 0, description: 'Test' }  // Zero amounts
        ],
        total_debit: 100,
        total_credit: 0,
        status: 'draft'
      };

      const result = validateZeroAmounts(transaction);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('has no amount');
    });
  });

  describe('Rule 4: Dual Amount Check', () => {
    test('should reject line with both debit and credit', () => {
      const transaction = {
        id: 'uuid',
        date: '2026-01-31',
        description: 'Test',
        lines: [
          { account_id: 'acc1', debit: 100, credit: 0, description: 'Test' },
          { account_id: 'acc2', debit: 50, credit: 50, description: 'Test' }  // Both amounts
        ],
        total_debit: 150,
        total_credit: 50,
        status: 'draft'
      };

      const result = validateDualAmounts(transaction);
      expect(result.valid).toBe(false);
      expect(result.error).toContain('cannot have both debit and credit');
    });
  });
});
```

### 6.2 Fuzzing Tests with Random Amounts
```typescript
import { random } from 'faker';

function generateRandomTransaction(count: number = 3): Transaction {
  const lines = [];
  let totalDebit = new Decimal(0);
  let totalCredit = new Decimal(0);

  for (let i = 0; i < count; i++) {
    const amount = new Decimal(random.number({ min: 1, max: 100000 }));

    if (i === count - 1) {
      // Balance the last line
      if (totalDebit.greaterThan(totalCredit)) {
        lines.push({
          account_id: `acc${i}`,
          debit: 0,
          credit: totalDebit.minus(totalCredit),
          description: 'Balancing line'
        });
      } else if (totalCredit.greaterThan(totalDebit)) {
        lines.push({
          account_id: `acc${i}`,
          debit: totalCredit.minus(totalDebit),
          credit: 0,
          description: 'Balancing line'
        });
      }
    } else {
      // Randomly assign debit or credit
      if (random.boolean()) {
        lines.push({
          account_id: `acc${i}`,
          debit: amount,
          credit: 0,
          description: 'Random line'
        });
        totalDebit = totalDebit.plus(amount);
      } else {
        lines.push({
          account_id: `acc${i}`,
          debit: 0,
          credit: amount,
          description: 'Random line'
        });
        totalCredit = totalCredit.plus(amount);
      }
    }
  }

  return {
    id: random.uuid(),
    date: random.date().toISOString(),
    description: 'Fuzzing test transaction',
    lines,
    total_debit: totalDebit,
    total_credit: totalCredit,
    status: 'draft'
  };
}

// Run fuzzing tests
describe('Fuzzing Tests', () => {
  test('should handle 100 random balanced transactions', () => {
    for (let i = 0; i < 100; i++) {
      const transaction = generateRandomTransaction();
      const result = validateTransaction(transaction);
      expect(result.valid).toBe(true);
    }
  });
});
```

### 6.3 Edge Cases: Rounding Errors, Large Numbers
```typescript
describe('Edge Cases', () => {
  test('should handle rounding errors within tolerance', () => {
    const transaction = {
      id: 'uuid',
      date: '2026-01-31',
      description: 'Rounding test',
      lines: [
        { account_id: 'acc1', debit: 0.01, credit: 0, description: 'Small amount' },
        { account_id: 'acc2', debit: 0, credit: 0.01, description: 'Small amount' }
      ],
      total_debit: 0.01,
      total_credit: 0.01,
      status: 'draft'
    };

    const result = validateBalance(transaction);
    expect(result.valid).toBe(true);
  });

  test('should handle maximum precision amounts', () => {
    const maxAmount = new Decimal('9999999999999.99');  // Max precision
    const transaction = {
      id: 'uuid',
      date: '2026-01-31',
      description: 'Large number test',
      lines: [
        { account_id: 'acc1', debit: maxAmount, credit: 0, description: 'Max amount' },
        { account_id: 'acc2', debit: 0, credit: maxAmount, description: 'Max amount' }
      ],
      total_debit: maxAmount,
      total_credit: maxAmount,
      status: 'draft'
    };

    const result = validateTransaction(transaction);
    expect(result.valid).toBe(true);
  });

  test('should reject amounts exceeding precision limits', () => {
    const overflowAmount = new Decimal('10000000000000.00');  // Exceeds 15,2 precision
    const transaction = {
      id: 'uuid',
      date: '2026-01-31',
      description: 'Overflow test',
      lines: [
        { account_id: 'acc1', debit: overflowAmount, credit: 0, description: 'Overflow' },
        { account_id: 'acc2', debit: 0, credit: overflowAmount, description: 'Overflow' }
      ],
      total_debit: overflowAmount,
      total_credit: overflowAmount,
      status: 'draft'
    };

    // This should be caught by database constraints
    expect(() => validateTransaction(transaction)).toThrow();
  });
});
```

---

## 7. API ENDPOINT

### 7.1 Validation Endpoint
```typescript
/**
 * Validates a transaction for double-entry compliance
 * @param {Transaction} transaction - The transaction to validate
 * @returns {Object} Validation result with valid flag and errors array
 */
app.post('/api/v1/transactions/validate', async (req, res) => {
  try {
    const { transaction } = req.body;

    // Validate input structure
    if (!transaction || !Array.isArray(transaction.lines)) {
      return res.status(400).json({
        valid: false,
        errors: ['Invalid transaction structure']
      });
    }

    // Run validation
    const result = await validateTransaction(transaction);

    if (result.valid) {
      res.status(200).json({
        valid: true,
        errors: []
      });
    } else {
      res.status(200).json({
        valid: false,
        errors: [result.error]
      });
    }
  } catch (error) {
    res.status(500).json({
      valid: false,
      errors: [`Internal validation error: ${error.message}`]
    });
  }
});
```

### 7.2 API Response Format
```json
{
  "valid": true,
  "errors": []
}
```

Or in case of validation failure:
```json
{
  "valid": false,
  "errors": [
    "Transaction out of balance by 100.00",
    "Line 2 cannot have both debit and credit"
  ]
}
```

---

**Document Classification:** Foundation Specification
**Review Cycle:** Annual review required or upon accounting standard changes
**Approval Authority:** Chief Financial Officer and Product Owner