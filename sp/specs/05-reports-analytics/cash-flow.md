# Cash Flow Statement Specification

## Overview
This specification defines the Cash Flow Statement report system for the accounting platform. The Cash Flow Statement shows the movement of cash and cash equivalents during a specific period, categorized into operating, investing, and financing activities.

## Cash Flow Categories

### Operating Activities
- Cash receipts from customers
- Cash payments to suppliers
- Cash payments for operating expenses
- Interest received/paid
- Income taxes paid
- Other operating cash flows

### Investing Activities
- Purchase of property, plant, and equipment
- Sale of property, plant, and equipment
- Purchase of investments
- Sale of investments
- Loans made to others
- Collections on loans

### Financing Activities
- Proceeds from borrowings
- Repayment of borrowings
- Issuance of shares
- Repurchase of shares
- Payment of dividends
- Other financing cash flows

## Report Structure

### Standard Cash Flow Statement Format
```
CASH FLOW STATEMENT
Period: [Start Date] to [End Date]

OPERATING ACTIVITIES
Cash received from customers:           $[XXX,XXX.XX]
Cash paid to suppliers:                $(XXX,XXX.XX)
Cash paid for operating expenses:      $(XXX,XXX.XX)
Interest paid:                         $(XXX,XXX.XX)
Income taxes paid:                     $(XXX,XXX.XX)
Other operating cash flows:            $[±XX,XXX.XX]
Net Cash from Operating Activities:    $[XXX,XXX.XX]

INVESTING ACTIVITIES
Purchase of equipment:                 $(XXX,XXX.XX)
Sale of investments:                   $[XX,XXX.XX]
Other investing cash flows:            $[±XX,XXX.XX]
Net Cash from Investing Activities:    $[±XX,XXX.XX]

FINANCING ACTIVITIES
Loans received:                        $[XX,XXX.XX]
Dividends paid:                        $(XX,XXX.XX)
Share repurchases:                     $(XX,XXX.XX)
Other financing cash flows:            $[±XX,XXX.XX]
Net Cash from Financing Activities:    $[±XX,XXX.XX]

NET CHANGE IN CASH:                    $[±XXX,XXX.XX]
Beginning Cash Balance:                $[XXX,XXX.XX]
Ending Cash Balance:                   $[XXX,XXX.XX]

Reconciliation of Net Income to Operating Cash Flow:
Net Income:                           $[XXX,XXX.XX]
Adjustments to reconcile net income to net cash from operating activities:
Depreciation:                         $[XX,XXX.XX]
Amortization:                         $[X,XXX.XX]
Changes in working capital:
  Accounts receivable:                $[±XX,XXX.XX]
  Inventory:                          $[±XX,XXX.XX]
  Accounts payable:                   $[±XX,XXX.XX]
  Accrued expenses:                   $[±X,XXX.XX]
Net Cash from Operating Activities:    $[XXX,XXX.XX]
```

## Direct vs Indirect Method

### Direct Method
- Reports major classes of gross cash receipts and payments
- Shows actual cash inflows and outflows
- More transparent but requires detailed cash tracking

### Indirect Method
- Starts with net income and adjusts for non-cash items
- Adjusts for changes in working capital
- Default method for the system
- Reconciles net income to cash flow from operations

### Method Selection
- Default: Indirect method
- Configurable to use direct method
- Automatic switching based on data availability
- Consistent method usage across periods

## Classification Logic

### Cash Flow Classification Algorithm
```
FOR each transaction in period DO
  IF transaction.account_type IN ('revenue', 'income') THEN
    IF customer_payment THEN
      classify as "Operating Activity - Cash received from customers"
    ELSE
      classify as "Operating Activity - Other operating cash inflow"
    END IF
  ELSE IF transaction.account_type IN ('expense', 'cost') THEN
    IF supplier_payment THEN
      classify as "Operating Activity - Cash paid to suppliers"
    ELSE IF operating_expense THEN
      classify as "Operating Activity - Cash paid for operating expenses"
    ELSE
      classify as "Operating Activity - Other operating cash outflow"
    END IF
  ELSE IF transaction.account_type IN ('asset') AND transaction.account_sub_type IN ('fixed_asset') THEN
    IF debit THEN
      classify as "Investing Activity - Purchase of equipment"
    ELSE
      classify as "Investing Activity - Sale of equipment"
    END IF
  ELSE IF transaction.account_type IN ('liability') AND transaction.account_sub_type IN ('long_term_debt') THEN
    IF credit THEN
      classify as "Financing Activity - Loans received"
    ELSE
      classify as "Financing Activity - Loan repayment"
    END IF
  ELSE IF transaction.account_type IN ('equity') AND transaction.account_sub_type IN ('dividend') THEN
    classify as "Financing Activity - Dividends paid"
  ELSE
    classify based on account mapping rules
  END IF
END FOR
```

### Account Mapping Rules
- Map specific accounts to cash flow categories
- Support for custom mappings per organization
- Default mappings based on account types
- Override capability for special cases

## AI Forecasting

### Forecasting AI Prompt
```
Based on the historical cash flow data for [Company Name] from [Start Date] to [End Date], analyze and predict:

1. Expected cash flow for the next month/quarter
2. Identify potential cash shortages and their timing
3. Analyze seasonal patterns in cash flow
4. Assess the impact of known upcoming payments/receipts
5. Provide probability ranges for forecasted amounts
6. Highlight risk factors that could affect cash flow
7. Suggest mitigation strategies for potential cash shortfalls
8. Predict cash position at end of forecast period

Consider: historical patterns, seasonal variations, business cycle, economic indicators, and any known future events.
```

### Forecasting Algorithm
```
FUNCTION forecast_cash_flow(historical_data, period_ahead)
  // Analyze historical patterns
  trend = calculate_trend(historical_data)
  seasonality = identify_seasonal_patterns(historical_data)
  volatility = measure_volatility(historical_data)

  // Calculate forecast for each category
  operating_forecast = forecast_category("operating", historical_data, trend, seasonality)
  investing_forecast = forecast_category("investing", historical_data, trend, seasonality)
  financing_forecast = forecast_category("financing", historical_data, trend, seasonality)

  // Combine forecasts
  net_change_forecast = operating_forecast + investing_forecast + financing_forecast
  ending_cash_forecast = current_cash + net_change_forecast

  // Calculate confidence intervals
  confidence_interval = calculate_confidence_interval(volatility, period_ahead)

  // Identify risks
  if ending_cash_forecast < minimum_cash_requirement THEN
    cash_shortage_alert = true
    shortage_timing = identify_shortage_timing(net_change_forecast)
  END IF

  RETURN {
    forecast: {
      operating: operating_forecast,
      investing: investing_forecast,
      financing: financing_forecast,
      net_change: net_change_forecast,
      ending_cash: ending_cash_forecast
    },
    confidence: confidence_interval,
    risks: identify_risks(historical_data),
    alerts: [cash_shortage_alert, shortage_timing]
  }
END FUNCTION
```

### Scenario Analysis
- Best case scenario (optimistic assumptions)
- Base case scenario (most likely assumptions)
- Worst case scenario (pessimistic assumptions)
- Sensitivity analysis for key variables
- Probability-weighted expected outcomes

## Report Template

### HTML Template Structure
```html
<div class="cash-flow-report">
  <div class="report-header">
    <h1>Cash Flow Statement</h1>
    <div class="report-period">Period: {{start_date}} to {{end_date}}</div>
    <div class="method">Method: {{method}}</div>
    <div class="company-info">{{company_name}}</div>
  </div>

  <div class="cash-flow-section">
    <h2>OPERATING ACTIVITIES</h2>
    <div class="activity-list">
      {{#each operating_activities}}
      <div class="activity-row">
        <span class="activity-description">{{description}}</span>
        <span class="amount {{amount_class}}">{{format_currency amount}}</span>
      </div>
      {{/each}}
    </div>
    <div class="section-total">
      <span>Net Cash from Operating Activities:</span>
      <span class="{{operating_net_class}}">{{format_currency operating_net}}</span>
    </div>
  </div>

  <div class="cash-flow-section">
    <h2>INVESTING ACTIVITIES</h2>
    <div class="activity-list">
      {{#each investing_activities}}
      <div class="activity-row">
        <span class="activity-description">{{description}}</span>
        <span class="amount {{amount_class}}">{{format_currency amount}}</span>
      </div>
      {{/each}}
    </div>
    <div class="section-total">
      <span>Net Cash from Investing Activities:</span>
      <span class="{{investing_net_class}}">{{format_currency investing_net}}</span>
    </div>
  </div>

  <div class="cash-flow-section">
    <h2>FINANCING ACTIVITIES</h2>
    <div class="activity-list">
      {{#each financing_activities}}
      <div class="activity-row">
        <span class="activity-description">{{description}}</span>
        <span class="amount {{amount_class}}">{{format_currency amount}}</span>
      </div>
      {{/each}}
    </div>
    <div class="section-total">
      <span>Net Cash from Financing Activities:</span>
      <span class="{{financing_net_class}}">{{format_currency financing_net}}</span>
    </div>
  </div>

  <div class="summary-section">
    <div class="summary-row">
      <span>Net Change in Cash:</span>
      <span class="{{net_change_class}}">{{format_currency net_change}}</span>
    </div>
    <div class="summary-row">
      <span>Beginning Cash Balance:</span>
      <span>{{format_currency beginning_cash}}</span>
    </div>
    <div class="summary-row highlight">
      <span>Ending Cash Balance:</span>
      <span class="{{ending_cash_class}}">{{format_currency ending_cash}}</span>
    </div>
  </div>

  {{#if indirect_method}}
  <div class="reconciliation-section">
    <h3>Reconciliation of Net Income to Operating Cash Flow</h3>
    <div class="reconciliation-details">
      {{#each reconciliation_items}}
      <div class="reconciliation-row">
        <span class="item-description">{{description}}</span>
        <span class="amount {{amount_class}}">{{format_currency amount}}</span>
      </div>
      {{/each}}
    </div>
  </div>
  {{/if}}

  {{#if forecast_available}}
  <div class="forecast-section">
    <h3>Cash Flow Forecast</h3>
    <div class="forecast-data">
      <div class="forecast-item">
        <span>Next Month Forecast:</span>
        <span class="{{forecast_class}}">{{format_currency forecast}}</span>
      </div>
      <div class="forecast-item">
        <span>Confidence Level:</span>
        <span>{{confidence_level}}%</span>
      </div>
      {{#if cash_shortage_risk}}
      <div class="alert-risk">
        ⚠️ Potential cash shortage detected in forecast
      </div>
      {{/if}}
    </div>
  </div>
  {{/if}}
</div>
```

## Security & Access Control

- Role-based access to cash flow reports
- Tenant isolation for multi-tenant environments
- Read-only access for non-financial roles
- Audit logging for report generation
- Data masking for sensitive cash information
- Period-specific access controls

## API Endpoints

### Report Generation
- `POST /api/reports/cash-flow` - Generate cash flow statement
- `GET /api/reports/cash-flow/{period}` - Get cash flow for specific period
- `POST /api/reports/cash-flow/method/{method}` - Generate with specific method

### Forecasting Operations
- `GET /api/reports/cash-flow/forecast` - Get cash flow forecast
- `POST /api/reports/cash-flow/scenario-analysis` - Perform scenario analysis
- `GET /api/reports/cash-flow/risk-assessment` - Get risk assessment

### Export Operations
- `POST /api/reports/cash-flow/export/excel` - Export to Excel
- `POST /api/reports/cash-flow/export/pdf` - Export to PDF
- `POST /api/reports/cash-flow/export/csv` - Export to CSV