# AI Compliance Agent Specification

## 1. Agent Purpose

The AI Compliance Agent is responsible for:
- Monitoring all transactions in real-time
- Enforcing accounting constitution and policies
- Flagging compliance violations automatically
- Blocking non-compliant operations
- Generating comprehensive compliance reports

## 2. Monitoring Rules

### A. Double-Entry Enforcement

```python
def validate_transaction(entry):
    """
    Validates that debits equal credits in journal entries
    """
    total_debits = sum(line.debit for line in entry.lines)
    total_credits = sum(line.credit for line in entry.lines)

    if abs(total_debits - total_credits) > 0.01:  # Allow minor rounding differences
        BLOCK_TRANSACTION()
        raise ComplianceError("Journal entry out of balance", {
            'total_debits': total_debits,
            'total_credits': total_credits,
            'difference': abs(total_debits - total_credits)
        })

    return True
```

### B. Period Lock Enforcement

```python
def check_period_lock(entry_date):
    """
    Ensures transactions are not posted to closed accounting periods
    """
    period = get_period(entry_date)
    if period.status == 'closed':
        BLOCK_TRANSACTION()
        raise ComplianceError(f"Cannot post to closed period {period.period_key}", {
            'period_status': period.status,
            'entry_date': entry_date,
            'period_start': period.start_date,
            'period_end': period.end_date
        })

    return True
```

### C. Account Validation

```python
def validate_accounts(lines):
    """
    Validates all accounts referenced in a transaction
    """
    for line in lines:
        account = get_account(line.account_id)
        if not account:
            BLOCK_TRANSACTION()
            raise ComplianceError(f"Account {line.account_id} does not exist", {
                'account_id': line.account_id,
                'line_number': line.line_number
            })

        if not account.is_active:
            BLOCK_TRANSACTION()
            raise ComplianceError(f"Account {account.code} is inactive", {
                'account_code': account.code,
                'account_status': account.status
            })

        if account.is_header:
            BLOCK_TRANSACTION()
            raise ComplianceError("Cannot post to header account", {
                'account_code': account.code,
                'account_type': account.type
            })

    return True
```

### D. Amount Validation

```python
def validate_amounts(lines):
    """
    Validates that all amounts are positive and reasonable
    """
    for line in lines:
        if line.debit < 0 or line.credit < 0:
            BLOCK_TRANSACTION()
            raise ComplianceError("Negative amounts not allowed", {
                'account_code': line.account_code,
                'debit': line.debit,
                'credit': line.credit
            })

        if line.debit == 0 and line.credit == 0:
            BLOCK_TRANSACTION()
            raise ComplianceError("Both debit and credit cannot be zero", {
                'account_code': line.account_code
            })

    return True
```

## 3. Anomaly Detection

### A. Unusual Amount Detection
```python
def detect_unusual_amounts(transaction_amount):
    """
    Detects amounts that are 3+ standard deviations from historical average
    """
    historical_avg = get_historical_average_amount(transaction_type)
    historical_std = get_historical_std_dev(transaction_type)

    if historical_avg and historical_std:
        z_score = abs(transaction_amount - historical_avg) / historical_std
        if z_score > 3:
            return {
                'anomaly_type': 'unusual_amount',
                'severity': 'high',
                'z_score': z_score,
                'historical_avg': historical_avg,
                'current_amount': transaction_amount
            }

    return None
```

### B. Duplicate Transaction Detection
```python
def detect_duplicate_transactions(amount, date, vendor, threshold_hours=24):
    """
    Detects potential duplicate transactions
    """
    recent_transactions = get_recent_transactions(date, threshold_hours)

    duplicates = []
    for tx in recent_transactions:
        if (tx.amount == amount and
            tx.vendor_id == vendor and
            abs((tx.date - date).total_seconds()) < threshold_hours * 3600):
            duplicates.append(tx)

    if duplicates:
        return {
            'anomaly_type': 'duplicate_transaction',
            'severity': 'medium',
            'duplicates_found': len(duplicates),
            'duplicate_details': [{'id': d.id, 'date': d.date} for d in duplicates]
        }

    return None
```

### C. Round Number Pattern Detection
```python
def detect_round_number_patterns(amount):
    """
    Detects round number patterns that may indicate fraud
    """
    rounded_amounts = [100, 500, 1000, 5000, 10000, 25000, 50000, 100000]

    for round_num in rounded_amounts:
        if abs(amount - round_num) <= 10:  # Within $10 of round number
            return {
                'anomaly_type': 'round_number_pattern',
                'severity': 'medium',
                'round_number': round_num,
                'difference': abs(amount - round_num)
            }

    return None
```

### D. Time-Based Anomaly Detection
```python
def detect_time_anomalies(transaction_datetime, business_hours=(9, 17)):
    """
    Detects transactions outside normal business hours
    """
    hour = transaction_datetime.hour

    if hour < business_hours[0] or hour >= business_hours[1]:
        return {
            'anomaly_type': 'after_hours_entry',
            'severity': 'low',
            'transaction_hour': hour,
            'business_hours': business_hours
        }

    if transaction_datetime.weekday() >= 5:  # Saturday or Sunday
        return {
            'anomaly_type': 'weekend_posting',
            'severity': 'low',
            'day_of_week': transaction_datetime.strftime('%A')
        }

    return None
```

## 4. Tax Compliance

### A. Tax Rate Validation
```python
def validate_tax_rates(transaction):
    """
    Validates that tax rates are appropriate for jurisdiction and item type
    """
    for line in transaction.lines:
        if line.tax_rate:
            expected_rate = get_expected_tax_rate(
                transaction.location,
                line.item_type,
                transaction.customer_type
            )

            if abs(line.tax_rate - expected_rate) > 0.001:  # 0.1% tolerance
                return {
                    'violation_type': 'tax_rate_mismatch',
                    'severity': 'high',
                    'expected_rate': expected_rate,
                    'actual_rate': line.tax_rate,
                    'difference': abs(line.tax_rate - expected_rate)
                }

    return None
```

### B. Tax Account Mapping Validation
```python
def validate_tax_account_mapping(transaction):
    """
    Ensures tax amounts are posted to correct tax liability accounts
    """
    for line in transaction.lines:
        if line.tax_amount > 0:
            expected_tax_account = get_expected_tax_account(
                line.tax_rate,
                transaction.location
            )

            if line.tax_account != expected_tax_account:
                return {
                    'violation_type': 'tax_account_mismatch',
                    'severity': 'high',
                    'expected_account': expected_tax_account,
                    'actual_account': line.tax_account
                }

    return None
```

### C. Tax Filing Deadline Tracking
```python
def check_tax_deadlines():
    """
    Monitors upcoming tax filing deadlines
    """
    upcoming_deadlines = get_upcoming_tax_deadlines(days_ahead=30)
    alerts = []

    for deadline in upcoming_deadlines:
        if deadline.days_until < 7:
            alerts.append({
                'alert_type': 'tax_deadline_approaching',
                'severity': 'medium' if deadline.days_until > 3 else 'high',
                'deadline': deadline.date,
                'days_remaining': deadline.days_until,
                'jurisdiction': deadline.jurisdiction,
                'form_type': deadline.form_type
            })

    return alerts
```

### D. Missing Tax Detection
```python
def detect_missing_tax(transaction):
    """
    Identifies taxable transactions without appropriate tax charges
    """
    for line in transaction.lines:
        if is_taxable_item(line.item_type) and line.tax_amount == 0:
            expected_tax = calculate_expected_tax(
                line.amount,
                transaction.location,
                line.item_type
            )

            if expected_tax > 0:
                return {
                    'violation_type': 'missing_tax_on_taxable_item',
                    'severity': 'high',
                    'item_type': line.item_type,
                    'expected_tax': expected_tax,
                    'actual_tax': 0
                }

    return None
```

## 5. Audit Trail Verification

### A. Change Logging Validation
```python
def verify_change_logging(transaction):
    """
    Ensures all changes to transactions are properly logged
    """
    if not transaction.change_log:
        return {
            'violation_type': 'missing_change_log',
            'severity': 'high',
            'transaction_id': transaction.id
        }

    # Verify that all changes have proper audit trail
    for change in transaction.change_log:
        if not change.user_id or not change.timestamp or not change.action:
            return {
                'violation_type': 'incomplete_audit_trail',
                'severity': 'medium',
                'transaction_id': transaction.id,
                'problematic_change': change.id
            }

    return None
```

### B. Document Number Gap Detection
```python
def detect_document_number_gaps():
    """
    Identifies gaps in sequential document numbering
    """
    recent_documents = get_recent_documents(limit=1000)
    gaps = []

    if len(recent_documents) > 1:
        prev_doc_num = recent_documents[0].document_number
        for doc in recent_documents[1:]:
            if doc.document_number != prev_doc_num - 1:
                gaps.append({
                    'gap_start': doc.document_number + 1,
                    'gap_end': prev_doc_num - 1,
                    'size': prev_doc_num - doc.document_number - 1
                })
            prev_doc_num = doc.document_number

    if gaps:
        return {
            'violation_type': 'document_number_gaps',
            'severity': 'high',
            'gaps_found': gaps
        }

    return None
```

### C. Voided Transaction Verification
```python
def verify_voided_transactions():
    """
    Ensures voided transactions are properly marked and reversed
    """
    voided_transactions = get_voided_transactions()
    violations = []

    for tx in voided_transactions:
        if not tx.void_reason:
            violations.append({
                'violation_type': 'voided_transaction_missing_reason',
                'severity': 'medium',
                'transaction_id': tx.id
            })

        # Check if reversal entry exists
        reversal_exists = check_for_reversal_entry(tx.id)
        if not reversal_exists:
            violations.append({
                'violation_type': 'voided_transaction_missing_reversal',
                'severity': 'high',
                'transaction_id': tx.id
            })

    if violations:
        return violations

    return None
```

### D. User Permission Verification
```python
def verify_user_permissions(operation, user, data):
    """
    Ensures users have appropriate permissions for operations
    """
    required_permission = get_required_permission(operation)

    if not user.has_permission(required_permission):
        return {
            'violation_type': 'insufficient_user_permissions',
            'severity': 'critical',
            'user_id': user.id,
            'required_permission': required_permission,
            'attempted_operation': operation
        }

    # Check amount limits
    if hasattr(data, 'amount') and data.amount > user.amount_limit:
        return {
            'violation_type': 'amount_limit_exceeded',
            'severity': 'high',
            'user_id': user.id,
            'user_limit': user.amount_limit,
            'transaction_amount': data.amount
        }

    return None
```

## 6. Agent Boundaries

### CAN Do
- Monitor ALL transactions in real-time
- Flag compliance violations automatically
- BLOCK non-compliant operations autonomously
- Generate comprehensive compliance reports
- Alert relevant stakeholders immediately

### CANNOT Do
- Approve compliance exceptions (human approval required)
- Modify compliance rules or constitutional policies
- Override fundamental accounting constitutional rules

### Requires Approval
- NEVER - operates with autonomous enforcement capabilities

## 7. Real-Time Monitoring

### Event-Driven Architecture
```
Transaction Created → Compliance Check Trigger → Rule Validation → Anomaly Detection → Decision Engine → Action Taken
```

### Webhook Configuration
- **Event Type**: Every transaction creation/modification
- **Trigger Point**: Before database commit
- **Response Time**: < 100ms for blocking decisions
- **Retry Logic**: 3 attempts with exponential backoff

### Instant Validation Process
1. **Receive Transaction Event**: Capture all transaction data
2. **Constitutional Validation**: Check against accounting constitution
3. **Anomaly Detection**: Run anomaly algorithms
4. **Tax Compliance Check**: Validate tax rules
5. **Audit Verification**: Ensure proper logging
6. **Decision Making**: Allow/block based on violations
7. **Action Execution**: Proceed or block transaction

## 8. Compliance Dashboard

### Key Metrics Displayed
- **Total Violations This Month**: Running count of compliance issues
- **Top Violation Types**: Most common violation categories
- **Risky Users**: Users with highest violation rates
- **Trend Analysis**: Compliance trends over time

### Dashboard Components
- **Real-time Alerts Panel**: Immediate violation notifications
- **Violation Severity Chart**: Critical vs. medium vs. low violations
- **Geographic Distribution**: Location-based compliance issues
- **User Activity Heatmap**: Transaction patterns by user

## 9. API Integration

### Endpoint: POST /api/v1/compliance/check

#### Request Body
```json
{
  "operation": "post_journal_entry",
  "data": {
    "entry_id": "je_12345",
    "date": "2024-01-15",
    "description": "Sales Invoice #INV-001",
    "lines": [
      {
        "account_id": "1200",
        "debit": 1150,
        "credit": 0
      },
      {
        "account_id": "4000",
        "debit": 0,
        "credit": 1000
      },
      {
        "account_id": "2100",
        "debit": 0,
        "credit": 150
      }
    ],
    "user_id": "usr_67890",
    "business_unit": "sales"
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "source_system": "journal_entry_ui"
}
```

#### Response Schema
```json
{
  "compliant": false,
  "violations": [
    {
      "rule": "period_lock",
      "severity": "critical",
      "message": "Cannot post to closed period 2024-01",
      "details": {
        "period_status": "closed",
        "entry_date": "2024-01-15",
        "period_start": "2024-01-01",
        "period_end": "2024-01-31"
      },
      "action": "blocked"
    }
  ],
  "action_taken": "transaction_blocked",
  "timestamp": "2024-01-15T10:30:00Z",
  "compliance_score": 0.0,
  "next_steps": [
    "Review period status",
    "Check with accounting team"
  ]
}
```

#### Alternative Response (Compliant)
```json
{
  "compliant": true,
  "violations": [],
  "action_taken": "transaction_allowed",
  "timestamp": "2024-01-15T10:30:00Z",
  "compliance_score": 1.0,
  "anomaly_indicators": [
    {
      "type": "round_number_pattern",
      "severity": "medium",
      "details": {
        "amount": 1000.00,
        "nearest_round": 1000
      }
    }
  ]
}
```

## 10. Event Monitoring Setup

### Event Subscriptions
- **Journal Entry Creation**: Monitor all new journal entries
- **Invoice Posting**: Track invoice creation and modifications
- **Payment Processing**: Monitor payment transactions
- **Account Changes**: Watch for account modifications
- **User Actions**: Track permission changes and access

### Monitoring Configuration
```python
EVENT_SUBSCRIPTIONS = {
    'journal_entry.created': ['validate_balance', 'check_period_lock', 'validate_accounts'],
    'journal_entry.updated': ['validate_balance', 'validate_accounts'],
    'invoice.posted': ['tax_compliance_check', 'duplicate_detection'],
    'payment.processed': ['duplicate_detection', 'amount_validation'],
    'account.modified': ['permission_verification', 'change_logging'],
    'user.permission_changed': ['audit_verification']
}
```

### Alert Configuration
- **Critical Violations**: Immediate notification to compliance officer
- **High-Risk Events**: Notification to department heads
- **Medium Issues**: Daily summary report
- **Low-Risk Patterns**: Weekly trend analysis

## 11. Compliance Report Templates

### Daily Compliance Report
```
Daily Compliance Summary - {date}

Total Transactions Processed: {count}
Violations Detected: {violations_count}
Compliance Rate: {rate}%

Top Violation Types:
- {type1}: {count1} occurrences
- {type2}: {count2} occurrences
- {type3}: {count3} occurrences

High-Risk Users:
- {user1}: {violations} violations
- {user2}: {violations} violations

Action Items:
- Review pending violations: {pending_count}
- Update user permissions: {permission_count}
```

### Monthly Compliance Report
```
Monthly Compliance Analysis - {month}

Key Metrics:
- Total transactions: {total}
- Violations detected: {violations}
- Average compliance score: {score}
- Escalated issues: {escalated}

Trend Analysis:
- Week-over-week violation change: {trend}%
- Improvement areas: {areas}
- Recurring issues: {recurring}

Recommendations:
- {recommendation1}
- {recommendation2}
- {recommendation3}
```

### Executive Compliance Dashboard
```
Executive Compliance Overview

Current Status: {status_icon}
Overall Compliance Score: {score}/100
Open Violations: {open_violations}
Pending Reviews: {pending_reviews}

Risk Assessment:
- High Risk: {high_risk_count}
- Medium Risk: {medium_risk_count}
- Low Risk: {low_risk_count}

Key Performance Indicators:
- Transaction volume: {volume}
- Average response time: {response_time}s
- Resolution rate: {resolution_rate}%
```

## 12. Technical Implementation

### Database Schema
- **compliance_logs**: Store all compliance checks and results
- **violations**: Track detected violations with severity
- **anomaly_patterns**: Store historical anomaly detection patterns
- **rules_configuration**: Maintain current compliance rule settings

### Performance Requirements
- **Response Time**: < 100ms for compliance decisions
- **Throughput**: Handle 1000+ transactions per minute
- **Availability**: 99.9% uptime for compliance monitoring
- **Scalability**: Horizontal scaling for high-volume periods

### Integration Points
- **GL System**: Monitor all general ledger entries
- **AP/AR Systems**: Track accounts payable/receivable
- **Payroll System**: Monitor payroll transactions
- **Tax System**: Validate tax calculations and filings