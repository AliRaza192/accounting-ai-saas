# Profit & Loss Report Specification

## Overview
This specification defines the Profit & Loss (P&L) report system for the accounting platform. The P&L report shows revenues, expenses, and profits over a specified period, providing insight into the financial performance of the business.

## Report Structure

### Standard P&L Hierarchy
```
Revenue
├── Sales Revenue
├── Service Revenue
└── Other Income
Total Revenue: $X

Cost of Goods Sold (COGS)
├── Direct Materials
├── Direct Labor
└── Manufacturing Overhead
Total COGS: $Y

Gross Profit: $X - $Y

Operating Expenses
├── Salaries & Wages
├── Rent
├── Utilities
├── Marketing & Advertising
├── Office Supplies
├── Insurance
└── Depreciation
Total Operating Expenses: $Z

Operating Income: Gross Profit - Operating Expenses

Other Income/Expenses
├── Interest Income
├── Interest Expense
└── Other Gains/Losses
Total Other Income/Expenses: $W

Net Income: Operating Income + Other Income/Expenses - Taxes
```

## Dynamic Period Selection

### Period Options
- **This Month**: Current month to date
- **This Quarter**: Current quarter to date
- **This Year**: Current fiscal year to date
- **Last Month**: Previous month
- **Last Quarter**: Previous quarter
- **Last Year**: Previous fiscal year
- **Custom Date Range**: User-defined start and end dates

### Comparison Options
- **Year-over-Year**: Current period vs. same period last year
- **Budget vs. Actual**: Actual results vs. budgeted amounts
- **Quarter-over-Quarter**: Current quarter vs. previous quarter
- **Month-over-Month**: Current month vs. previous month

## SQL Query Logic

### Base Query Structure
```sql
WITH period_data AS (
  SELECT
    je.gl_account_id,
    a.account_code,
    a.account_name,
    a.account_type,
    a.parent_account_id,
    SUM(jel.debit_amount - jel.credit_amount) AS net_amount
  FROM transactions.journal_entries je
  JOIN transactions.journal_entry_lines jel ON je.id = jel.journal_entry_id
  JOIN chart_of_accounts a ON jel.account_id = a.id
  WHERE je.transaction_date BETWEEN :start_date AND :end_date
    AND je.tenant_id = :tenant_id
    AND je.status = 'posted'
  GROUP BY je.gl_account_id, a.account_code, a.account_name, a.account_type, a.parent_account_id
),
hierarchical_accounts AS (
  SELECT
    *,
    CASE
      WHEN account_type = 'revenue' THEN 'Revenue'
      WHEN account_type = 'cogs' THEN 'Cost of Goods Sold'
      WHEN account_type = 'expense' THEN 'Operating Expenses'
      ELSE 'Other'
    END AS major_category
  FROM period_data
)
SELECT
  major_category,
  account_name,
  account_code,
  SUM(net_amount) AS amount
FROM hierarchical_accounts
GROUP BY major_category, account_name, account_code
ORDER BY
  CASE major_category
    WHEN 'Revenue' THEN 1
    WHEN 'Cost of Goods Sold' THEN 2
    WHEN 'Operating Expenses' THEN 3
    ELSE 4
  END,
  account_name;
```

### Comparison Query Structure
```sql
WITH current_period AS (
  -- Current period data (same as base query)
),
prior_period AS (
  SELECT
    je.gl_account_id,
    a.account_code,
    a.account_name,
    a.account_type,
    a.parent_account_id,
    SUM(jel.debit_amount - jel.credit_amount) AS net_amount
  FROM transactions.journal_entries je
  JOIN transactions.journal_entry_lines jel ON je.id = jel.journal_entry_id
  JOIN chart_of_accounts a ON jel.account_id = a.id
  WHERE je.transaction_date BETWEEN :comparison_start_date AND :comparison_end_date
    AND je.tenant_id = :tenant_id
    AND je.status = 'posted'
  GROUP BY je.gl_account_id, a.account_code, a.account_name, a.account_type, a.parent_account_id
)
SELECT
  cp.major_category,
  cp.account_name,
  cp.account_code,
  cp.amount AS current_amount,
  pp.net_amount AS comparison_amount,
  (cp.amount - pp.net_amount) AS variance,
  CASE
    WHEN pp.net_amount != 0 THEN ((cp.amount - pp.net_amount) / pp.net_amount) * 100
    ELSE NULL
  END AS variance_percentage
FROM current_period cp
LEFT JOIN prior_period pp ON cp.gl_account_id = pp.gl_account_id
ORDER BY ...;
```

## Report Template

### HTML Template Structure
```html
<div class="pnl-report">
  <div class="report-header">
    <h1>Profit & Loss Statement</h1>
    <div class="report-period">Period: {{period}}</div>
    <div class="company-info">{{company_name}}</div>
  </div>

  <div class="pnl-section">
    <h2>Revenue</h2>
    <div class="account-list">
      {{#each revenue_accounts}}
      <div class="account-row">
        <span class="account-name">{{name}}</span>
        <span class="amount">{{format_currency amount}}</span>
      </div>
      {{/each}}
    </div>
    <div class="section-total">
      <span>Total Revenue:</span>
      <span>{{format_currency total_revenue}}</span>
    </div>
  </div>

  <div class="pnl-section">
    <h2>Cost of Goods Sold</h2>
    <div class="account-list">
      {{#each cogs_accounts}}
      <div class="account-row">
        <span class="account-name">{{name}}</span>
        <span class="amount">{{format_currency amount}}</span>
      </div>
      {{/each}}
    </div>
    <div class="section-total">
      <span>Total COGS:</span>
      <span>{{format_currency total_cogs}}</span>
    </div>
  </div>

  <div class="section-total">
    <span>Gross Profit:</span>
    <span>{{format_currency gross_profit}}</span>
  </div>

  <div class="pnl-section">
    <h2>Operating Expenses</h2>
    <div class="account-list">
      {{#each operating_expense_accounts}}
      <div class="account-row">
        <span class="account-name">{{name}}</span>
        <span class="amount negative">{{format_currency amount}}</span>
      </div>
      {{/each}}
    </div>
    <div class="section-total">
      <span>Total Operating Expenses:</span>
      <span>{{format_currency total_operating_expenses}}</span>
    </div>
  </div>

  <div class="section-total">
    <span>Net Income:</span>
    <span>{{format_currency net_income}}</span>
  </div>
</div>
```

## Excel Export Format

### Worksheet Structure
```
Sheet 1: P&L Summary
A1: Company Name
A2: Profit & Loss Statement
A3: Period: [Selected Period]
A5: Account Category
B5: Account Name
C5: Amount
A6: Revenue
A7:   Sales Revenue
B7:   [Account Name]
C7:   [Amount]
...
A[n]: Total Revenue
C[n]: [Total Amount]

Sheet 2: Detail Transactions
A1: Account
B1: Date
C1: Description
D1: Debit
E1: Credit
F1: Balance
```

### Excel Formatting Specifications
- Currency formatting for all monetary values
- Bold headers and totals
- Conditional formatting for negative values
- Column width optimization
- Filterable headers
- Professional color scheme

## Drill-Down Capability

### Transaction Drill-Down
- Click on any account name/amount to view underlying transactions
- Modal or new page showing detailed journal entries
- Filter options for the drill-down view
- Export options for the detailed view

### Export Options
- Excel format with formulas and formatting
- PDF format for professional presentation
- CSV format for data analysis
- Print-optimized layout

### Email Scheduling
- Configure automatic email delivery
- Select recipients and frequency
- Customize email content
- Include multiple periods/comparisons in email

## Variance Analysis (AI)

### AI Analysis Prompt
```
Analyze the Profit & Loss statement for [Company Name] for the period [Date Range] compared to [Comparison Period]. Identify and explain:

1. Significant variances (>10% or >$X threshold) in revenue and expense categories
2. Trends in account performance
3. Potential causes for unusual fluctuations
4. Risk areas requiring management attention
5. Opportunities for improvement
6. Predictions for future performance based on current trends

Focus on material changes and provide actionable insights. Consider seasonal patterns, business growth, and external economic factors in your analysis.
```

### Variance Detection Algorithm
```
FOR each account in P&L DO
  IF ABS(current_amount - comparison_amount) / ABS(comparison_amount) > variance_threshold OR
     ABS(current_amount - comparison_amount) > dollar_threshold THEN
    Flag account for review
    Calculate percentage variance
    Determine significance level
    Suggest explanation based on historical patterns
  END IF
END FOR

IF revenue_variance > threshold THEN
  Trigger revenue analysis
ELSE IF expense_variance > threshold THEN
  Trigger expense analysis
END IF
```

### Trend Prediction Model
- Historical data analysis
- Seasonal adjustment factors
- Growth rate calculations
- Confidence intervals for predictions

## Report Parameters

### Required Parameters
- **Tenant ID**: For multi-tenant data isolation
- **Start Date**: Beginning of reporting period
- **End Date**: End of reporting period
- **Comparison Type**: Type of comparison (YoY, Budget, etc.)
- **Format**: Output format (HTML, PDF, Excel)

### Optional Parameters
- **Include Budget**: Whether to include budget columns
- **Grouping Level**: How to group accounts (detailed, summary)
- **Currency**: Currency for reporting
- **Columns**: Which columns to display

## Security & Access Control

- Role-based access to P&L reports
- Tenant isolation for multi-tenant environments
- Read-only access for non-financial roles
- Audit logging for report generation
- Data masking for sensitive information

## API Endpoints

### Report Generation
- `POST /api/reports/pnl` - Generate P&L report
- `GET /api/reports/pnl/templates` - Get available templates
- `GET /api/reports/pnl/schedule` - Get scheduled reports

### Export Operations
- `POST /api/reports/pnl/export/excel` - Export to Excel
- `POST /api/reports/pnl/export/pdf` - Export to PDF
- `POST /api/reports/pnl/export/csv` - Export to CSV

### Drill-Down Operations
- `GET /api/reports/pnl/{report_id}/drilldown/{account_id}` - Get transaction details
- `GET /api/reports/pnl/{report_id}/variance-analysis` - Get AI variance analysis

### Schedule Management
- `POST /api/reports/pnl/schedule` - Schedule report delivery
- `PUT /api/reports/pnl/schedule/{schedule_id}` - Update schedule
- `DELETE /api/reports/pnl/schedule/{schedule_id}` - Delete schedule