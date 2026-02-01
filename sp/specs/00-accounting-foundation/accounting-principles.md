# ACCOUNTING PRINCIPLES SPECIFICATION - AccountingAI Pro

**Specification Version:** 1.0
**Effective Date:** January 31, 2026
**Classification:** Foundation Specification

---

## OVERVIEW

This specification defines the fundamental accounting principles that govern all financial operations within the AccountingAI Pro system. These principles ensure compliance with international accounting standards and provide a solid foundation for all financial reporting and processing.

---

## 1. FUNDAMENTAL CONCEPTS

### 1.1 Accrual Basis Accounting
The system operates on an accrual basis, recognizing revenues when earned and expenses when incurred, regardless of cash exchange timing.

**Implementation:**
- Revenue recognition at point of delivery/service completion
- Expense recognition when obligation is incurred
- Accrued income and expenses tracked separately
- Deferral of prepaid expenses and unearned revenue

**Example:**
```
Service provided on Dec 15, 2025, paid on Jan 5, 2026:
Dec 15: Revenue recognized, accounts receivable increased
Jan 5: Cash received, accounts receivable decreased
```

### 1.2 Going Concern Assumption
The system assumes the business entity will continue operating indefinitely unless evidence suggests otherwise.

**System Behavior:**
- Asset values based on ongoing operations
- Depreciation calculated over useful life
- Long-term obligations treated as continuing commitments
- No liquidation value assumptions applied

### 1.3 Consistency Principle
Accounting methods and procedures must remain consistent across reporting periods.

**System Enforcement:**
- Policy selection stored at entity level
- Change tracking and approval required
- Disclosure requirements for changes
- Comparative reporting with prior periods

### 1.4 Materiality Concept
Items that could influence economic decisions must be disclosed, regardless of dollar amount.

**Materiality Thresholds:**
- Absolute threshold: $1,000 or 1% of total assets, whichever is lower
- Relative threshold: Items affecting >5% of net income
- Qualitative factors: Industry-specific considerations

### 1.5 Conservatism Principle
When faced with uncertainty, choose methods that are least likely to overstate assets and income.

**Application:**
- Provision for doubtful accounts
- Lower of cost or market inventory valuation
- Contingent liability recognition
- Revenue recognition only when assured

---

## 2. ACCOUNTING EQUATION

### 2.1 Fundamental Equation
```
Assets = Liabilities + Equity
```

### 2.2 System Requirements
- **Real-time Validation**: Balance must be maintained after every transaction
- **Continuous Monitoring**: System alerts if equation becomes unbalanced
- **Automated Correction**: Suggest corrections for imbalances

### 2.3 Validation Pseudocode
```
FUNCTION validate_accounting_equation():
    total_assets = SUM(asset_accounts)
    total_liabilities = SUM(liability_accounts)
    total_equity = SUM(equity_accounts)

    difference = total_assets - (total_liabilities + total_equity)

    IF ABS(difference) > 0.01 THEN  // 0.01 tolerance for rounding
        LOG_ERROR("Accounting equation imbalance: " + difference)
        ALERT_BALANCE_SHEET_REVIEW()
        RETURN FALSE
    ELSE
        RETURN TRUE
    END IF
END FUNCTION

FUNCTION enforce_balance_during_transaction(transaction):
    PRE_TRANSACTION_BALANCE = validate_accounting_equation()

    EXECUTE_TRANSACTION(transaction)

    POST_TRANSACTION_BALANCE = validate_accounting_equation()

    IF NOT POST_TRANSACTION_BALANCE THEN
        ROLLBACK_TRANSACTION()
        LOG_CRITICAL_ERROR("Transaction caused accounting equation imbalance")
        THROW_ACCOUNTING_VIOLATION_EXCEPTION()
    END IF
END FUNCTION
```

### 2.4 Balance Maintenance Rules
- All journal entries must maintain equation balance
- System rejects unbalanced entries
- Trial balance validation at period end
- Real-time monitoring of account relationships

---

## 3. REVENUE RECOGNITION

### 3.1 Core Principles
Revenue is recognized when earned, not when cash is received.

**Criteria for Recognition:**
- Persuasive evidence of arrangement exists
- Delivery has occurred or services rendered
- Seller's price is fixed or determinable
- Collectibility is reasonably assured

### 3.2 Revenue Recognition Patterns
- **Point-in-time**: Revenue recognized when control transferred
- **Over-time**: Revenue recognized as performance obligation satisfied
- **Contract-based**: Multiple performance obligations handled separately

### 3.3 Deferred Revenue Handling
```
// Example: Annual subscription received in advance
Cash Dr. $1,200
Deferred Revenue Cr. $1,200

// Monthly recognition
Deferred Revenue Dr. $100
Revenue Cr. $100
```

### 3.4 Multi-Element Arrangements
- Identify separate performance obligations
- Allocate transaction price based on standalone selling prices
- Recognize revenue for each element when satisfied

### 3.5 Validation Pseudocode
```
FUNCTION validate_revenue_recognition(transaction):
    IF transaction.type == "REVENUE" THEN
        // Verify recognition criteria met
        criteria_met = CHECK_RECOGNITION_CRITERIA(transaction)

        IF NOT criteria_met THEN
            LOG_WARNING("Revenue recognition criteria not met")
            REQUIRE_MANUAL_APPROVAL()
        END IF

        // Check for deferral requirements
        IF transaction.recognized_immediately AND
           transaction.payment_received_in_advance THEN
            LOG_ERROR("Immediate recognition of deferred revenue")
            THROW_REVENUE_RECOGNITION_VIOLATION()
        END IF
    END IF
END FUNCTION

FUNCTION check_deferral_requirements(revenue_transaction):
    IF revenue_transaction.service_period > 1_month THEN
        IF NOT revenue_transaction.has_deferral_schedule() THEN
            CREATE_DEFERRED_REVENUE_SCHEDULE(revenue_transaction)
        END IF
    END IF
END FUNCTION
```

---

## 4. EXPENSE MATCHING

### 4.1 Matching Principle
Expenses must be matched to the revenues they helped generate, or to the period in which they were incurred.

### 4.2 Matching Methods
- **Direct Matching**: Expenses directly tied to revenue generation
- **Systematic Allocation**: Expenses allocated over time periods
- **Immediate Recognition**: Expenses with no future benefit

### 4.3 Period Allocation Rules
- Salaries and wages: Allocated to period worked
- Rent and utilities: Allocated to period benefiting
- Insurance: Allocated over coverage period
- Supplies: Expensed when consumed

### 4.4 Prepaid/Accrued Handling
```
// Prepaid expense example
Prepaid Insurance Dr. $1,200
Cash Cr. $1,200

// Monthly amortization
Insurance Expense Dr. $100
Prepaid Insurance Cr. $100

// Accrued expense example
Expense Dr. $500
Accrued Liability Cr. $500
```

### 4.5 Validation Pseudocode
```
FUNCTION validate_expense_matching(expense_transaction):
    related_revenue = FIND_RELATED_REVENUE(expense_transaction)

    IF related_revenue EXISTS THEN
        revenue_period = GET_ACCOUNTING_PERIOD(related_revenue)
        expense_period = GET_ACCOUNTING_PERIOD(expense_transaction)

        IF ABS(DIFF_IN_MONTHS(revenue_period, expense_period)) > 1 THEN
            LOG_WARNING("Expense-revenue mismatch: " +
                       revenue_period + " vs " + expense_period)
        END IF
    END IF

    // Validate allocation periods
    IF expense_transaction.requires_allocation THEN
        allocation_periods = CALCULATE_ALLOCATION_PERIODS(expense_transaction)
        IF allocation_periods != expense_transaction.allocation_schedule THEN
            LOG_ERROR("Incorrect expense allocation schedule")
            THROW_EXPENSE_MATCHING_VIOLATION()
        END IF
    END IF
END FUNCTION
```

---

## 5. REPORTING STANDARDS

### 5.1 GAAP Support
- Generally Accepted Accounting Principles compliance
- FASB guidance implementation
- Industry-specific requirements
- SEC reporting compliance

### 5.2 IFRS Support
- International Financial Reporting Standards compliance
- IASB guidance implementation
- Consolidation requirements
- Fair value measurements

### 5.3 Configurable Per Tenant
```
tenant_settings = {
    accounting_standard: "GAAP" | "IFRS" | "LOCAL",
    reporting_currency: "USD" | "EUR" | ...,
    fiscal_year_start: "01-01" | "04-01" | ...,
    depreciation_methods: [...],
    inventory_valuation: "FIFO" | "LIFO" | "WeightedAverage"
}
```

### 5.4 Standard Reports by Framework
- **GAAP Reports**: Income Statement, Balance Sheet, Cash Flow Statement
- **IFRS Reports**: Similar structure with IFRS-specific disclosures
- **Local GAAP**: Adapted to local requirements

### 5.5 Validation Pseudocode
```
FUNCTION validate_reporting_standard_compliance(tenant_id, transaction):
    tenant_standard = GET_TENANT_ACCOUNTING_STANDARD(tenant_id)

    IF tenant_standard == "GAAP" THEN
        compliance_check = VALIDATE_GAAP_COMPLIANCE(transaction)
    ELSE IF tenant_standard == "IFRS" THEN
        compliance_check = VALIDATE_IFRS_COMPLIANCE(transaction)
    ELSE
        compliance_check = VALIDATE_LOCAL_COMPLIANCE(transaction, tenant_standard)
    END IF

    IF NOT compliance_check.passed THEN
        LOG_ERROR("Transaction violates " + tenant_standard + " standards")
        THROW_STANDARDS_COMPLIANCE_VIOLATION()
    END IF
END FUNCTION

FUNCTION generate_standard_reports(tenant_id, period):
    tenant_standard = GET_TENANT_ACCOUNTING_STANDARD(tenant_id)

    SELECT CASE tenant_standard:
        WHEN "GAAP":
            reports = [
                GENERATE_GAAP_INCOME_STATEMENT(period),
                GENERATE_GAAP_BALANCE_SHEET(period),
                GENERATE_GAAP_CASH_FLOW(period)
            ]
        WHEN "IFRS":
            reports = [
                GENERATE_IFRS_INCOME_STATEMENT(period),
                GENERATE_IFRS_BALANCE_SHEET(period),
                GENERATE_IFRS_CASH_FLOW(period)
            ]
        ELSE:
            reports = GENERATE_LOCAL_STANDARD_REPORTS(period, tenant_standard)
    END SELECT

    RETURN reports
END FUNCTION
```

---

## 6. ACCOUNTING POLICIES

### 6.1 Depreciation Methods
- **Straight-Line Method**: Equal annual depreciation
- **Declining Balance**: Accelerated depreciation
- **Units of Production**: Based on usage

```
// Straight-line example
annual_depreciation = (cost - salvage_value) / useful_life

// Declining balance example
depreciation_rate = 2 / useful_life
annual_depreciation = book_value * depreciation_rate
```

### 6.2 Inventory Valuation Methods
- **FIFO (First-In, First-Out)**: Oldest costs assigned to COGS
- **LIFO (Last-In, First-Out)**: Newest costs assigned to COGS
- **Weighted Average**: Average cost assigned to all units

### 6.3 Bad Debt Estimation
- **Percentage of Sales Method**: Fixed percentage of credit sales
- **Aging Method**: Different percentages by age of receivables
- **Specific Identification**: Individual account assessment

### 6.4 Policy Selection and Validation
```
policy_settings = {
    depreciation_method: "STRAIGHT_LINE" | "DECLINING_BALANCE" | "UNITS_OF_PRODUCTION",
    inventory_valuation: "FIFO" | "LIFO" | "WEIGHTED_AVERAGE",
    bad_debt_method: "PERCENTAGE_OF_SALES" | "AGING_METHOD" | "SPECIFIC_IDENTIFICATION",
    bad_debt_percentage: DECIMAL_VALUE
}
```

### 6.5 Validation Pseudocode
```
FUNCTION validate_accounting_policy_consistency(tenant_id, transaction):
    policies = GET_TENANT_POLICIES(tenant_id)

    // Validate depreciation consistency
    IF transaction.affects_depreciable_assets THEN
        IF policies.depreciation_method != transaction.depreciation_method THEN
            LOG_ERROR("Depreciation method inconsistency")
            THROW_POLICY_VIOLATION()
        END IF
    END IF

    // Validate inventory valuation consistency
    IF transaction.affects_inventory THEN
        IF policies.inventory_valuation != transaction.inventory_method THEN
            LOG_ERROR("Inventory valuation method inconsistency")
            THROW_POLICY_VIOLATION()
        END IF
    END IF

    // Validate bad debt method consistency
    IF transaction.affects_allowance_for_doubtful_accounts THEN
        IF policies.bad_debt_method != transaction.bad_debt_method THEN
            LOG_ERROR("Bad debt estimation method inconsistency")
            THROW_POLICY_VIOLATION()
        END IF
    END IF
END FUNCTION

FUNCTION handle_policy_change(tenant_id, old_policies, new_policies):
    IF old_policies != new_policies THEN
        LOG_WARNING("Accounting policy change detected")

        // Generate re-statement warnings
        GENERATE_RESTATEMENT_WARNINGS(tenant_id, old_policies, new_policies)

        // Create comparison reports
        comparison_report = GENERATE_POLICY_CHANGE_COMPARISON(
            tenant_id,
            old_policies,
            new_policies
        )

        SEND_TO_AUDIT_TRAIL(comparison_report)

        // Notify stakeholders
        NOTIFY_STAKEHOLDERS_POLICY_CHANGE(tenant_id, comparison_report)
    END IF
END FUNCTION
```

---

## VALIDATION RULES

### System Enforced Rules
- Accounting standard compliance validation
- Policy consistency checks
- Equation balance verification
- Revenue recognition criteria validation

### Change Management
- Policy change tracking and approval
- Restatement warnings for material changes
- Historical data impact analysis

### Comparison Reports
- Pre-change vs post-change financials
- Policy impact quantification
- Stakeholder communication reports

---

**Document Classification:** Foundation Specification
**Review Cycle:** Annual review required or upon accounting standard changes
**Approval Authority:** Chief Financial Officer and Product Owner