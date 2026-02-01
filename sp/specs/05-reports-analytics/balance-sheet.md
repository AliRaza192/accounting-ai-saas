# Balance Sheet Report Specification

## Overview
This specification defines the Balance Sheet report system for the accounting platform. The Balance Sheet report shows the financial position of the business at a specific point in time, displaying assets, liabilities, and equity in accordance with the fundamental accounting equation: Assets = Liabilities + Equity.

## Report Structure

### Standard Balance Sheet Hierarchy
```
ASSETS
Current Assets
├── Cash & Bank
├── Accounts Receivable
├── Inventory
├── Prepaid Expenses
└── Other Current Assets
Total Current Assets: $X

Fixed Assets
├── Property, Plant & Equipment
├── Vehicles
├── Furniture & Fixtures
├── Less: Accumulated Depreciation
└── Net Fixed Assets
Total Fixed Assets: $Y

Other Assets
├── Long-term Investments
├── Intangible Assets
└── Other Long-term Assets
Total Other Assets: $Z

Total Assets: $X + $Y + $Z = $A

LIABILITIES
Current Liabilities
├── Accounts Payable
├── Accrued Expenses
├── Short-term Loans
├── Current Portion of Long-term Debt
└── Other Current Liabilities
Total Current Liabilities: $B

Long-term Liabilities
├── Long-term Loans Payable
├── Bonds Payable
└── Other Long-term Liabilities
Total Long-term Liabilities: $C

Total Liabilities: $B + $C = $D

EQUITY
├── Share Capital
├── Retained Earnings
├── Current Year Profit
└── Other Equity Components
Total Equity: $E

Total Liabilities + Equity: $D + $E = $F

Balance Check: $A (Total Assets) = $F (Total Liabilities + Equity) ✓
```

## SQL Query

### Base Balance Sheet Query
```sql
WITH account_balances AS (
  SELECT
    a.id AS account_id,
    a.account_code,
    a.account_name,
    a.account_type,
    a.account_sub_type,
    a.parent_account_id,
    COALESCE(SUM(
      CASE
        WHEN je.transaction_date <= :end_date THEN jel.debit_amount - jel.credit_amount
        ELSE 0
      END
    ), 0) AS balance
  FROM chart_of_accounts a
  LEFT JOIN transactions.journal_entry_lines jel ON a.id = jel.account_id
  LEFT JOIN transactions.journal_entries je ON jel.journal_entry_id = je.id
  WHERE a.tenant_id = :tenant_id
    AND (je.status = 'posted' OR je.id IS NULL)
    AND a.is_active = true
  GROUP BY a.id, a.account_code, a.account_name, a.account_type, a.account_sub_type, a.parent_account_id
),
balance_sheet_structure AS (
  SELECT
    account_id,
    account_code,
    account_name,
    account_type,
    account_sub_type,
    balance,
    CASE
      WHEN account_type IN ('asset') AND account_sub_type IN ('current_asset') THEN 'Current Assets'
      WHEN account_type IN ('asset') AND account_sub_type IN ('fixed_asset') THEN 'Fixed Assets'
      WHEN account_type IN ('asset') AND account_sub_type IN ('other_asset') THEN 'Other Assets'
      WHEN account_type IN ('liability') AND account_sub_type IN ('current_liability') THEN 'Current Liabilities'
      WHEN account_type IN ('liability') AND account_sub_type IN ('long_term_liability') THEN 'Long-term Liabilities'
      WHEN account_type IN ('equity') THEN 'Equity'
      ELSE 'Other'
    END AS major_category,
    CASE
      WHEN account_type IN ('asset') THEN 'Assets'
      WHEN account_type IN ('liability') THEN 'Liabilities'
      WHEN account_type IN ('equity') THEN 'Equity'
      ELSE 'Other'
    END AS section
  FROM account_balances
  WHERE balance != 0 OR account_type IN ('asset', 'liability', 'equity')
)
SELECT
  section,
  major_category,
  account_name,
  account_code,
  balance,
  CASE
    WHEN account_type = 'asset' AND balance < 0 THEN balance * -1
    WHEN account_type IN ('liability', 'equity') AND balance > 0 THEN balance * -1
    ELSE balance
  END AS normalized_balance
FROM balance_sheet_structure
ORDER BY
  CASE section
    WHEN 'Assets' THEN 1
    WHEN 'Liabilities' THEN 2
    WHEN 'Equity' THEN 3
    ELSE 4
  END,
  CASE major_category
    WHEN 'Current Assets' THEN 1
    WHEN 'Fixed Assets' THEN 2
    WHEN 'Other Assets' THEN 3
    WHEN 'Current Liabilities' THEN 4
    WHEN 'Long-term Liabilities' THEN 5
    WHEN 'Equity' THEN 6
    ELSE 7
  END,
  account_name;
```

### Comparative Balance Sheet Query
```sql
WITH current_period AS (
  -- Current period data (same as base query)
),
prior_period AS (
  SELECT
    a.id AS account_id,
    a.account_code,
    a.account_name,
    a.account_type,
    a.account_sub_type,
    a.parent_account_id,
    COALESCE(SUM(
      CASE
        WHEN je.transaction_date <= :prior_end_date THEN jel.debit_amount - jel.credit_amount
        ELSE 0
      END
    ), 0) AS balance
  FROM chart_of_accounts a
  LEFT JOIN transactions.journal_entry_lines jel ON a.id = jel.account_id
  LEFT JOIN transactions.journal_entries je ON jel.journal_entry_id = je.id
  WHERE a.tenant_id = :tenant_id
    AND (je.status = 'posted' OR je.id IS NULL)
    AND a.is_active = true
  GROUP BY a.id, a.account_code, a.account_name, a.account_type, a.account_sub_type, a.parent_account_id
)
SELECT
  cp.section,
  cp.major_category,
  cp.account_name,
  cp.current_balance,
  pp.balance AS prior_balance,
  (cp.current_balance - pp.balance) AS variance,
  CASE
    WHEN pp.balance != 0 THEN ((cp.current_balance - pp.balance) / pp.balance) * 100
    ELSE NULL
  END AS variance_percentage
FROM current_period cp
LEFT JOIN prior_period pp ON cp.account_id = pp.account_id
ORDER BY ...;
```

## Balance Validation

### Balance Equation Validation
```
IF ABS(total_assets - (total_liabilities + total_equity)) > tolerance_value THEN
  status = "Unbalanced"
  difference = total_assets - (total_liabilities + total_equity)
  alert_message = "Balance sheet does not balance. Difference: " + format_currency(difference)
ELSE
  status = "Balanced"
  difference = 0
  alert_message = "Balance sheet balances correctly"
END IF
```

### Validation Tolerance
- Standard tolerance: $0.01 (to account for rounding differences)
- Configurable tolerance for large organizations
- Critical alerts for differences exceeding tolerance

### Validation Process
1. Calculate total assets
2. Calculate total liabilities
3. Calculate total equity
4. Compare Assets vs (Liabilities + Equity)
5. Generate alert if imbalance detected
6. Highlight specific accounts causing imbalance

## Comparative Reports

### This Year vs Last Year
- Side-by-side comparison of balance sheet positions
- Variance analysis showing changes between periods
- Percentage change calculations
- Visual indicators for significant changes

### Month-over-Month
- Rolling monthly comparisons
- Trend analysis for asset/liability movements
- Seasonal adjustment factors
- 12-month moving averages

## Report Template

### HTML Template Structure
```html
<div class="balance-sheet-report">
  <div class="report-header">
    <h1>Balance Sheet</h1>
    <div class="report-date">As of: {{date}}</div>
    <div class="company-info">{{company_name}}</div>
  </div>

  <div class="report-body">
    <!-- ASSETS SECTION -->
    <div class="section">
      <h2>ASSETS</h2>

      <div class="subsection">
        <h3>Current Assets</h3>
        <div class="account-list">
          {{#each current_assets}}
          <div class="account-row">
            <span class="account-name">{{name}}</span>
            <span class="amount">{{format_currency amount}}</span>
          </div>
          {{/each}}
        </div>
        <div class="subsection-total">
          <span>Total Current Assets:</span>
          <span>{{format_currency total_current_assets}}</span>
        </div>
      </div>

      <div class="subsection">
        <h3>Fixed Assets</h3>
        <div class="account-list">
          {{#each fixed_assets}}
          <div class="account-row">
            <span class="account-name">{{name}}</span>
            <span class="amount">{{format_currency amount}}</span>
          </div>
          {{/each}}
        </div>
        <div class="subsection-total">
          <span>Total Fixed Assets:</span>
          <span>{{format_currency total_fixed_assets}}</span>
        </div>
      </div>

      <div class="subsection">
        <h3>Other Assets</h3>
        <div class="account-list">
          {{#each other_assets}}
          <div class="account-row">
            <span class="account-name">{{name}}</span>
            <span class="amount">{{format_currency amount}}</span>
          </div>
          {{/each}}
        </div>
        <div class="subsection-total">
          <span>Total Other Assets:</span>
          <span>{{format_currency total_other_assets}}</span>
        </div>
      </div>

      <div class="section-total">
        <span><strong>Total Assets:</strong></span>
        <span><strong>{{format_currency total_assets}}</strong></span>
      </div>
    </div>

    <!-- LIABILITIES SECTION -->
    <div class="section">
      <h2>LIABILITIES</h2>

      <div class="subsection">
        <h3>Current Liabilities</h3>
        <div class="account-list">
          {{#each current_liabilities}}
          <div class="account-row">
            <span class="account-name">{{name}}</span>
            <span class="amount">{{format_currency amount}}</span>
          </div>
          {{/each}}
        </div>
        <div class="subsection-total">
          <span>Total Current Liabilities:</span>
          <span>{{format_currency total_current_liabilities}}</span>
        </div>
      </div>

      <div class="subsection">
        <h3>Long-term Liabilities</h3>
        <div class="account-list">
          {{#each long_term_liabilities}}
          <div class="account-row">
            <span class="account-name">{{name}}</span>
            <span class="amount">{{format_currency amount}}</span>
          </div>
          {{/each}}
        </div>
        <div class="subsection-total">
          <span>Total Long-term Liabilities:</span>
          <span>{{format_currency total_long_term_liabilities}}</span>
        </div>
      </div>

      <div class="section-total">
        <span><strong>Total Liabilities:</strong></span>
        <span><strong>{{format_currency total_liabilities}}</strong></span>
      </div>
    </div>

    <!-- EQUITY SECTION -->
    <div class="section">
      <h2>EQUITY</h2>
      <div class="account-list">
        {{#each equity_accounts}}
        <div class="account-row">
          <span class="account-name">{{name}}</span>
          <span class="amount">{{format_currency amount}}</span>
        </div>
        {{/each}}
      </div>
      <div class="section-total">
        <span><strong>Total Equity:</strong></span>
        <span><strong>{{format_currency total_equity}}</strong></span>
      </div>
    </div>

    <!-- BALANCE CHECK -->
    <div class="balance-check">
      <div class="balance-equation">
        <span>Total Liabilities + Equity:</span>
        <span>{{format_currency sum_liabilities_equity}}</span>
      </div>
      <div class="balance-status {{balance_class}}">
        Balance Check: {{balance_result}}
        {{#if difference}}(Difference: {{format_currency difference}}){{/if}}
      </div>
    </div>
  </div>
</div>
```

## Security & Access Control

- Role-based access to balance sheet reports
- Tenant isolation for multi-tenant environments
- Read-only access for non-financial roles
- Audit logging for report generation
- Data masking for sensitive information
- Period-specific access controls

## API Endpoints

### Report Generation
- `POST /api/reports/balance-sheet` - Generate balance sheet report
- `GET /api/reports/balance-sheet/{date}` - Get balance sheet for specific date
- `POST /api/reports/balance-sheet/comparative` - Generate comparative balance sheet

### Validation Operations
- `GET /api/reports/balance-sheet/{date}/validate` - Validate balance sheet balance
- `POST /api/reports/balance-sheet/{date}/revalidate` - Force revalidation

### Export Operations
- `POST /api/reports/balance-sheet/export/excel` - Export to Excel
- `POST /api/reports/balance-sheet/export/pdf` - Export to PDF
- `POST /api/reports/balance-sheet/export/csv` - Export to CSV