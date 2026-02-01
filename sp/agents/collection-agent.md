# AI Accounts Receivable Collection Agent Specification

## 1. Agent Purpose

The AI Accounts Receivable Collection Agent is designed to:
- Monitor aging receivables continuously
- Send automated payment reminders at appropriate intervals
- Escalate overdue accounts according to established rules
- Optimize the collection process to improve cash flow

## 2. Aging Buckets

### A. Classification Structure
```
Current (0-30 days): Recently issued invoices, due soon
30-60 days overdue: First stage of delinquency
60-90 days overdue: Second stage, requires attention
90+ days overdue: Severe delinquency, escalation required
```

### B. Aging Calculation Logic
```python
def calculate_aging_buckets(customer_invoices, current_date):
    """
    Calculates aging buckets for customer invoices
    """
    from datetime import timedelta

    aging_buckets = {
        'current': {'invoices': [], 'total': 0},
        '30_days': {'invoices': [], 'total': 0},
        '60_days': {'invoices': [], 'total': 0},
        '90_plus': {'invoices': [], 'total': 0}
    }

    for invoice in customer_invoices:
        days_overdue = (current_date - invoice.due_date).days

        if days_overdue <= 0:
            bucket = 'current'
        elif days_overdue <= 30:
            bucket = '30_days'
        elif days_overdue <= 60:
            bucket = '60_days'
        else:
            bucket = '90_plus'

        aging_buckets[bucket]['invoices'].append(invoice)
        aging_buckets[bucket]['total'] += invoice.balance

    return aging_buckets
```

## 3. Automated Reminders

### A. Day 1 (Invoice Sent)
**Subject:** Invoice #INV-001 - Payment Due Jan 31

**Body:**
```
Hi [Customer Name],

Thank you for your business! Invoice #INV-001 for $1,000
is due on Jan 31, 2024.

[View Invoice Button] [Pay Now Button]

Payment methods accepted:
- Bank transfer
- Credit card
- Check

Questions? Simply reply to this email.

Best regards,
[Company Name] Accounts Receivable Team
```

### B. Day 25 (5 days before due)
**Subject:** Reminder: Invoice #INV-001 Due Soon

**Body:**
```
Dear [Customer Name],

This is a friendly reminder that invoice #INV-001 for $1,000
is due in 5 days (Jan 31, 2024).

[Pay Now Button]

If you have any concerns about payment, please reach out
to us before the due date.

Thank you for your prompt attention.

[Company Name] Team
```

### C. Day 35 (5 days overdue)
**Subject:** OVERDUE: Invoice #INV-001 - Immediate Attention Required

**Body:**
```
URGENT: Overdue Invoice Notice

Invoice #INV-001 is now 5 days overdue. Please remit
payment immediately to avoid late fees.

Amount Due: $1,050 (includes $50 late fee)

[Pay Now Button]

We value our relationship and would appreciate your
prompt attention to this matter.

Contact us immediately if you need payment assistance.

[Company Name] Accounts Receivable
```

### D. Day 65 (35 days overdue)
**Subject:** URGENT: Account Escalation - Invoice #INV-001

**Body:**
```
ACCOUNT ESCALATION NOTICE

Your account is seriously overdue. We will escalate to
collections if payment is not received within 7 days.

Please contact us immediately to arrange payment.

Amount Due: $1,050
Due Date: Jan 31, 2024
Days Overdue: 35

We still hope to resolve this amicably. Please call
[Phone Number] or email [Email] today.

[Company Name] Collections Department
```

### E. Email Template System
```python
EMAIL_TEMPLATES = {
    'invoice_sent': {
        'subject': 'Invoice #{invoice_number} - Payment Due {due_date}',
        'body': '''
Hi {customer_name},

Thank you for your business! Invoice #{invoice_number} for ${amount}
is due on {due_date}.

[View Invoice Button] [Pay Now Button]

Payment methods accepted:
- Bank transfer
- Credit card
- Check

Questions? Simply reply to this email.

Best regards,
{company_name} Accounts Receivable Team
        ''',
        'schedule_days': 0
    },
    'pre_due_reminder': {
        'subject': 'Reminder: Invoice #{invoice_number} Due Soon',
        'body': '''
Dear {customer_name},

This is a friendly reminder that invoice #{invoice_number} for ${amount}
is due in 5 days ({due_date}).

[Pay Now Button]

If you have any concerns about payment, please reach out
to us before the due date.

Thank you for your prompt attention.

{company_name} Team
        ''',
        'schedule_days': 25
    },
    'overdue_initial': {
        'subject': 'OVERDUE: Invoice #{invoice_number} - Immediate Attention Required',
        'body': '''
URGENT: Overdue Invoice Notice

Invoice #{invoice_number} is now 5 days overdue. Please remit
payment immediately to avoid late fees.

Amount Due: ${total_amount_due} (includes ${late_fee} late fee)

[Pay Now Button]

We value our relationship and would appreciate your
prompt attention to this matter.

Contact us immediately if you need payment assistance.

{company_name} Accounts Receivable
        ''',
        'schedule_days': 35
    },
    'severe_delinquency': {
        'subject': 'URGENT: Account Escalation - Invoice #{invoice_number}',
        'body': '''
ACCOUNT ESCALATION NOTICE

Your account is seriously overdue. We will escalate to
collections if payment is not received within 7 days.

Please contact us immediately to arrange payment.

Amount Due: ${amount_due}
Due Date: {due_date}
Days Overdue: {days_overdue}

We still hope to resolve this amicably. Please call
{phone_number} or email {email} today.

{company_name} Collections Department
        ''',
        'schedule_days': 65
    }
}
```

## 4. Escalation Rules

### A. Escalation Hierarchy
```python
ESCALATION_RULES = {
    30: {
        'action': 'assign_to_senior_accountant',
        'notification': 'senior_accountant',
        'customer_action': 'monitor_closely',
        'timeline': 'within_3_business_days'
    },
    60: {
        'action': 'cfo_notification',
        'notification': 'cfo',
        'customer_action': 'hold_new_orders',
        'timeline': 'immediate'
    },
    90: {
        'action': 'collections_referral',
        'notification': 'collections_agency',
        'customer_action': 'suspend_services',
        'timeline': 'within_1_business_day'
    }
}
```

### B. Escalation Workflow
```python
def process_escalation(customer_id, days_overdue):
    """
    Processes escalation based on days overdue
    """
    if days_overdue >= 90:
        return handle_90_day_escalation(customer_id)
    elif days_overdue >= 60:
        return handle_60_day_escalation(customer_id)
    elif days_overdue >= 30:
        return handle_30_day_escalation(customer_id)

    return {'status': 'no_escalation_needed'}

def handle_30_day_escalation(customer_id):
    """
    Handles 30-day overdue escalation
    """
    # Assign to senior accountant
    assign_to_senior_accountant(customer_id)

    # Send notification
    notify_senior_accountant(customer_id)

    # Update customer status
    update_customer_status(customer_id, 'monitor_closely')

    return {
        'action': 'assigned_to_senior',
        'notification_sent': True,
        'customer_updated': True
    }

def handle_60_day_escalation(customer_id):
    """
    Handles 60-day overdue escalation
    """
    # Notify CFO
    notify_cfo(customer_id)

    # Hold new orders
    suspend_new_orders(customer_id)

    # Update customer status
    update_customer_status(customer_id, 'restricted')

    return {
        'action': 'cfo_notified',
        'new_orders_suspended': True,
        'customer_updated': True
    }

def handle_90_day_escalation(customer_id):
    """
    Handles 90-day overdue escalation
    """
    # Refer to collections agency
    refer_to_collections(customer_id)

    # Suspend services
    suspend_customer_services(customer_id)

    # Update customer status
    update_customer_status(customer_id, 'collections_pending')

    return {
        'action': 'collections_referred',
        'services_suspended': True,
        'customer_updated': True
    }
```

## 5. Payment Plan Suggestions

### A. Payment Plan Logic
```python
def suggest_payment_plans(customer_id, outstanding_amount, days_overdue):
    """
    Suggests payment plans based on customer history and amount owed
    """
    customer_history = get_customer_payment_history(customer_id)

    # Calculate discount based on payment history
    loyalty_discount = calculate_loyalty_discount(customer_history)

    plans = []

    # Option 1: Full payment with discount
    if loyalty_discount > 0:
        discounted_amount = outstanding_amount * (1 - loyalty_discount)
        plans.append({
            'option': 'full_payment_discount',
            'description': f'Full payment now → {loyalty_discount*100:.0f}% discount (${discounted_amount:.2f})',
            'amount': discounted_amount,
            'installments': 1,
            'timeline': 'immediate',
            'confidence': 0.7 if days_overdue < 60 else 0.3
        })

    # Option 2: Installment plan
    if outstanding_amount > 1000:
        num_installments = min(6, max(2, int(outstanding_amount / 2000)))
        installment_amount = outstanding_amount / num_installments

        plans.append({
            'option': 'installment_plan',
            'description': f'{num_installments} monthly installments (${installment_amount:.2f} each)',
            'amount': outstanding_amount,
            'installments': num_installments,
            'timeline': f'{num_installments} months',
            'confidence': 0.8
        })

    # Option 3: Partial payment
    if days_overdue >= 60:
        partial_payment = outstanding_amount * 0.5
        remaining_payment = outstanding_amount - partial_payment

        plans.append({
            'option': 'partial_then_full',
            'description': f'{partial_payment:.2f} now, {remaining_payment:.2f} in 30 days',
            'amount': outstanding_amount,
            'installments': 2,
            'timeline': '30 days',
            'confidence': 0.6
        })

    return {
        'customer_id': customer_id,
        'outstanding_amount': outstanding_amount,
        'suggested_plans': plans,
        'recommended_option': select_best_plan(plans, customer_history)
    }

def calculate_loyalty_discount(customer_history):
    """
    Calculates loyalty discount based on payment history
    """
    if not customer_history:
        return 0.0

    # Calculate based on average days to pay and payment consistency
    avg_days_to_pay = customer_history.get('avg_days_to_pay', 60)
    payment_consistency = customer_history.get('payment_consistency', 0.5)

    if avg_days_to_pay < 30 and payment_consistency > 0.8:
        return 0.05  # 5% discount for excellent history
    elif avg_days_to_pay < 45 and payment_consistency > 0.6:
        return 0.02  # 2% discount for good history
    else:
        return 0.0  # No discount for poor history
```

### B. Example Payment Plan
**Customer owes:** $10,000 (60 days overdue)

**Suggested Payment Plan:**
```
Option 1: Full payment now → 5% discount ($9,500)
Option 2: 3 monthly installments ($3,333.33 each)
Option 3: $5,000 now, $5,000 in 30 days

[Accept Plan Button]
```

## 6. Agent Boundaries

### CAN Do
- Monitor aging receivables automatically
- Send scheduled payment reminders
- Escalate accounts per established rules
- Suggest payment plans to customers
- Track and maintain communication history

### CANNOT Do
- Write off debts (requires CFO approval)
- Modify payment terms without proper authorization
- Share customer data with external parties

### Requires Approval
- **Custom payment plans**: Beyond standard options
- **Debt write-offs**: Any account write-off decision

## 7. Metrics Tracking

### A. Key Performance Indicators
```python
def calculate_collection_metrics():
    """
    Calculates key collection metrics
    """
    metrics = {}

    # Days Sales Outstanding (DSO)
    metrics['dso'] = calculate_dso()

    # Collection Rate
    metrics['collection_rate'] = calculate_collection_rate()

    # Aging Trend
    metrics['aging_trend'] = calculate_aging_trend()

    # Reminder Effectiveness
    metrics['reminder_effectiveness'] = calculate_reminder_effectiveness()

    # Customer Retention Rate
    metrics['customer_retention'] = calculate_customer_retention()

    return metrics

def calculate_dso():
    """
    Calculates Days Sales Outstanding
    """
    from datetime import datetime

    total_receivables = get_total_receivables()
    daily_sales = get_daily_sales_average()

    if daily_sales > 0:
        return total_receivables / daily_sales
    else:
        return 0

def calculate_collection_rate():
    """
    Calculates collection rate
    """
    collected_amount = get_collected_amount_last_period()
    total_due = get_total_due_last_period()

    if total_due > 0:
        return (collected_amount / total_due) * 100
    else:
        return 0
```

### B. Metric Definitions
- **DSO (Days Sales Outstanding)**: Average days to collect payment
- **Collection Rate**: Percentage of receivables collected
- **Aging Trend**: Movement between aging buckets over time
- **Reminder Effectiveness**: Success rate of different reminder types
- **Customer Retention**: Percentage of customers who continue paying

## 8. API Integration

### Endpoint: POST /api/v1/ai/collections/analyze

#### Request Body
```json
{
  "customer_id": "cust_123",
  "include_history": true,
  "as_of_date": "2024-01-31",
  "user_id": "user_456"
}
```

#### Response Schema
```json
{
  "customer_id": "cust_123",
  "analysis_date": "2024-01-31",
  "total_outstanding": 15000,
  "aging": {
    "current": {
      "amount": 5000,
      "invoices": ["inv_001", "inv_002"]
    },
    "30_days": {
      "amount": 4000,
      "invoices": ["inv_003"]
    },
    "60_days": {
      "amount": 3000,
      "invoices": ["inv_004"]
    },
    "90_plus": {
      "amount": 3000,
      "invoices": ["inv_005"]
    }
  },
  "payment_history": {
    "average_days_to_pay": 45,
    "reliability_score": 0.7,
    "payment_frequency": "monthly",
    "last_payment_date": "2023-12-15"
  },
  "recommended_actions": [
    {
      "action": "send_reminder",
      "priority": "high",
      "reason": "$3K is 90+ days overdue",
      "scheduled_date": "2024-02-01"
    },
    {
      "action": "payment_plan",
      "terms": "3 monthly installments",
      "confidence": 0.8
    },
    {
      "action": "escalate_to_senior",
      "priority": "medium",
      "reason": "60+ days overdue pattern",
      "target": "senior_accountant"
    }
  ],
  "communication_history": [
    {
      "date": "2024-01-15",
      "type": "email",
      "subject": "Overdue Invoice Reminder",
      "status": "opened"
    }
  ],
  "collection_score": 0.65,
  "next_scheduled_action": {
    "type": "phone_call",
    "date": "2024-02-05",
    "priority": "high"
  }
}
```

## 9. Escalation Workflow

### A. Complete Workflow Process
```
1. Monitor Aging Daily
   ↓
2. Identify Overdue Accounts
   ↓
3. Check Customer Payment History
   ↓
4. Apply Escalation Rules
   ↓
5. Send Appropriate Communication
   ↓
6. Track Response and Update Status
   ↓
7. Continue Monitoring Until Resolved
```

### B. Workflow Implementation
```python
def run_daily_collection_process():
    """
    Runs the daily collection process
    """
    # Get all customers with outstanding invoices
    customers_with_overdue = get_overdue_customers()

    for customer in customers_with_overdue:
        # Calculate aging buckets
        aging = calculate_aging_buckets(customer.invoices, datetime.now())

        # Determine maximum days overdue
        max_overdue = get_max_days_overdue(aging)

        # Process escalation if needed
        escalation_result = process_escalation(customer.id, max_overdue)

        # Send appropriate reminders
        reminder_result = send_appropriate_reminders(customer, aging)

        # Suggest payment plans if needed
        if max_overdue >= 30:
            payment_plan = suggest_payment_plans(
                customer.id,
                aging['total_outstanding'],
                max_overdue
            )

        # Update customer record
        update_customer_collection_record(customer.id, {
            'last_contact_date': datetime.now(),
            'escalation_level': escalation_result,
            'payment_plan_offered': payment_plan
        })

def send_appropriate_reminders(customer, aging):
    """
    Sends appropriate reminders based on aging
    """
    results = []

    for bucket, data in aging.items():
        if data['total'] > 0:
            # Determine which template to use based on days overdue
            template = get_appropriate_template(bucket)

            if template:
                result = send_email_reminder(
                    customer.email,
                    template['subject'],
                    template['body'],
                    customer
                )

                results.append(result)

    return results
```

## 10. Metrics Dashboard Specification

### A. Dashboard Components
- **Aging Summary Widget**: Visual representation of aging buckets
- **DSO Trend Chart**: Historical DSO performance
- **Collection Rate Meter**: Current collection effectiveness
- **Customer Health Score**: Individual customer risk assessment
- **Escalation Queue**: Accounts requiring immediate attention
- **Communication Effectiveness**: Success rates by communication type

### B. Dashboard Data Model
```python
DASHBOARD_LAYOUT = {
    'top_row': [
        {'widget': 'aging_summary', 'position': 0, 'size': 'large'},
        {'widget': 'dso_chart', 'position': 1, 'size': 'medium'},
        {'widget': 'collection_rate', 'position': 2, 'size': 'small'}
    ],
    'middle_row': [
        {'widget': 'customer_health', 'position': 0, 'size': 'large'},
        {'widget': 'escalation_queue', 'position': 1, 'size': 'medium'}
    ],
    'bottom_row': [
        {'widget': 'communication_effectiveness', 'position': 0, 'size': 'large'},
        {'widget': 'trending_customers', 'position': 1, 'size': 'medium'}
    ]
}
```

### C. Real-time Updates
- Live aging bucket updates
- Instant escalation notifications
- Real-time communication tracking
- Dynamic customer risk scoring

## 11. Performance Requirements

### A. Processing Requirements
- **Daily aging updates**: Complete within 1 hour
- **Reminder sending**: Process within 15 minutes of trigger
- **Escalation processing**: Immediate notification within 5 minutes
- **Customer analysis**: Complete within 30 seconds

### B. Accuracy Requirements
- **Aging calculation**: 100% accuracy
- **Payment plan suggestions**: 90% effectiveness rate
- **Escalation triggers**: 99.9% accuracy
- **Communication tracking**: 100% reliability

### C. Availability Requirements
- **Uptime**: 99.9% availability
- **Backup systems**: Redundant processing capabilities
- **Disaster recovery**: Restore within 4 hours
- **Data retention**: 7-year retention for compliance