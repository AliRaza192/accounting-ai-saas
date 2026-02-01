# AI Bank Reconciliation Agent Specification

## 1. Agent Purpose

The AI Bank Reconciliation Agent is designed to:
- Automatically match bank transactions to General Ledger entries
- Suggest appropriate handling for unmatched items
- Flag potential discrepancies and errors
- Significantly speed up the bank reconciliation process

## 2. Matching Algorithms

### A. Exact Match (Confidence: 100%)

```python
def exact_match(bank_tx, gl_entries):
    """
    Finds exact matches based on amount and date
    """
    for gl_entry in gl_entries:
        if (bank_tx.amount == gl_entry.amount and
            bank_tx.date == gl_entry.date and
            bank_tx.transaction_type == gl_entry.transaction_type):
            return {
                'matched_entry': gl_entry,
                'confidence': 1.0,
                'match_type': 'exact',
                'match_reason': 'Amount and date exactly match'
            }
    return None
```

### B. Fuzzy Date Match (Confidence: 90%)

```python
def fuzzy_date_match(bank_tx, gl_entries, date_tolerance_days=3):
    """
    Matches based on amount with date tolerance
    """
    from datetime import timedelta

    for gl_entry in gl_entries:
        date_diff = abs((bank_tx.date - gl_entry.date).days)

        if (bank_tx.amount == gl_entry.amount and
            date_diff <= date_tolerance_days):
            return {
                'matched_entry': gl_entry,
                'confidence': 0.9,
                'match_type': 'fuzzy_date',
                'match_reason': f'Amount exact, date within {date_diff} days',
                'date_difference': date_diff
            }
    return None
```

### C. Description Match (Confidence: 85%)

```python
def description_match(bank_tx, gl_entries, similarity_threshold=0.8, amount_tolerance=0.01):
    """
    Uses fuzzy string matching on descriptions with amount tolerance
    """
    from difflib import SequenceMatcher

    for gl_entry in gl_entries:
        # Calculate description similarity
        similarity = SequenceMatcher(None,
                                  bank_tx.description.lower(),
                                  gl_entry.description.lower()).ratio()

        # Check if amounts are close enough
        amount_diff = abs(bank_tx.amount - gl_entry.amount)
        is_amount_close = amount_diff <= (abs(gl_entry.amount) * amount_tolerance)

        if similarity > similarity_threshold and is_amount_close:
            return {
                'matched_entry': gl_entry,
                'confidence': 0.85,
                'match_type': 'description',
                'match_reason': f'Description similarity {similarity:.2f}, amounts close',
                'description_similarity': similarity,
                'amount_difference': amount_diff
            }
    return None
```

### D. Split Transaction Match (Confidence: 75%)

```python
def split_transaction_match(bank_tx, gl_entries, tolerance=0.01):
    """
    Matches one bank transaction to multiple GL entries that sum to the same amount
    """
    # Find combinations of GL entries that sum to bank transaction amount
    from itertools import combinations

    for r in range(2, len(gl_entries) + 1):
        for combo in combinations(gl_entries, r):
            combo_sum = sum(entry.amount for entry in combo)
            difference = abs(bank_tx.amount - combo_sum)

            if difference <= (abs(bank_tx.amount) * tolerance):
                return {
                    'matched_entries': list(combo),
                    'confidence': 0.75,
                    'match_type': 'split_transaction',
                    'match_reason': f'{len(combo)} GL entries sum to bank amount',
                    'difference': difference
                }
    return None
```

### E. Advanced Matching with Priority

```python
def advanced_matching(bank_tx, gl_entries):
    """
    Runs all matching algorithms in priority order
    """
    # Priority order: exact, fuzzy date, description, split
    match_algorithms = [
        ('exact', exact_match),
        ('fuzzy_date', fuzzy_date_match),
        ('description', description_match),
        ('split', split_transaction_match)
    ]

    for alg_name, alg_func in match_algorithms:
        if alg_name == 'exact':
            result = alg_func(bank_tx, gl_entries)
        elif alg_name == 'fuzzy_date':
            result = alg_func(bank_tx, gl_entries)
        elif alg_name == 'description':
            result = alg_func(bank_tx, gl_entries)
        elif alg_name == 'split':
            result = alg_func(bank_tx, gl_entries)

        if result:
            return result

    return None
```

## 3. Confidence Scoring Logic

### A. Base Confidence Calculation
```python
def calculate_confidence(match_result):
    """
    Calculates final confidence score based on match type and additional factors
    """
    base_confidence = match_result.get('confidence', 0.0)

    # Adjust confidence based on additional factors
    adjustments = []

    # Transaction frequency factor
    if is_frequent_transaction_type(match_result['matched_entry']):
        adjustments.append(0.05)

    # Amount significance factor
    if is_significant_amount(match_result['matched_entry'].amount):
        adjustments.append(-0.05)  # Lower confidence for large amounts

    # Historical accuracy factor
    if has_high_accuracy_history(match_result['matched_entry']):
        adjustments.append(0.03)

    final_confidence = base_confidence + sum(adjustments)
    return min(max(final_confidence, 0.0), 1.0)  # Clamp between 0 and 1
```

### B. Confidence Thresholds
- **Auto-Match**: Confidence > 95% (automatically reconcile)
- **Suggest**: 80-95% (suggest for user review)
- **Flag**: < 80% (require manual attention)

## 4. Unmatched Item Handling

### A. Bank Fee (Not Recorded)

**Example:**
Bank Transaction: -$25 "Monthly Service Charge"

**Suggestion:**
Create journal entry:
- Debit: Bank Fees Expense $25
- Credit: Bank Account $25

**Implementation:**
```python
def identify_bank_fees(unmatched_bank_txs):
    """
    Identifies potential bank fees in unmatched transactions
    """
    fee_indicators = [
        'service charge', 'monthly fee', 'overdraft', 'atm fee',
        'wire transfer fee', 'account maintenance', 'nsf fee'
    ]

    bank_fees = []
    for tx in unmatched_bank_txs:
        if any(indicator in tx.description.lower() for indicator in fee_indicators):
            suggestion = {
                'transaction': tx,
                'suggested_action': 'create_journal_entry',
                'journal_entry': {
                    'description': f'Bank fee for {tx.description}',
                    'lines': [
                        {
                            'account': '5200',  # Bank Fees Expense
                            'debit': abs(tx.amount),
                            'credit': 0
                        },
                        {
                            'account': tx.bank_account_code,
                            'debit': 0,
                            'credit': abs(tx.amount)
                        }
                    ]
                },
                'confidence': 0.9,
                'reason': 'Bank fee identified by description'
            }
            bank_fees.append(suggestion)

    return bank_fees
```

### B. Deposit in Transit

**Example:**
GL Entry: Deposit $5,000 (Jan 31)
Bank: No matching transaction

**Suggestion:**
Mark as "Deposit in Transit"
Will clear when bank processes it

**Implementation:**
```python
def identify_deposit_in_transit(unmatched_gl_entries, bank_statement_end_date):
    """
    Identifies deposits in transit
    """
    deposits_in_transit = []

    for entry in unmatched_gl_entries:
        if entry.amount > 0 and entry.date > bank_statement_end_date:
            suggestion = {
                'entry': entry,
                'suggested_action': 'mark_as_deposit_in_transit',
                'reason': f'Deposit dated after bank statement period ({entry.date})',
                'confidence': 0.95,
                'estimated_clearing_date': estimate_clearing_date(entry)
            }
            deposits_in_transit.append(suggestion)

    return deposits_in_transit
```

### C. Outstanding Check

**Example:**
GL Entry: Check #1234 for $1,200
Bank: No matching debit

**Suggestion:**
Mark as "Outstanding Check"

**Implementation:**
```python
def identify_outstanding_checks(unmatched_gl_entries, bank_statement_end_date):
    """
    Identifies outstanding checks
    """
    outstanding_checks = []

    check_indicators = ['check', 'chk', '#', 'payment']

    for entry in unmatched_gl_entries:
        if entry.amount < 0 and any(indicator in entry.description.lower() for indicator in check_indicators):
            suggestion = {
                'entry': entry,
                'suggested_action': 'mark_as_outstanding_check',
                'reason': f'Check payment not yet cleared by bank',
                'confidence': 0.9,
                'estimated_clearing_date': estimate_clearing_date(entry)
            }
            outstanding_checks.append(suggestion)

    return outstanding_checks
```

## 5. Agent Boundaries

### CAN Do
- Auto-match transactions with confidence >95%
- Suggest matches with 80-95% confidence for user review
- Flag discrepancies and potential errors
- Recommend appropriate journal entries for bank fees

### CANNOT Do
- Force final reconciliation without user approval
- Modify bank statement data directly
- Auto-create journal entries without explicit user approval

### Requires Approval
- **Adjustments > $100**: Require supervisor approval
- **Matches with confidence < 95%**: Require user verification
- **Suspicious activity**: Require investigation

## 6. Reconciliation Workflow

```
1. Import Bank Statement
   ↓
2. Agent Auto-Matches (>95% confidence)
   ↓
3. Suggest Matches (80-95%) for Review
   ↓
4. Flag Unmatched Items for Investigation
   ↓
5. User Reviews and Approves/Rejects Suggestions
   ↓
6. Generate Reconciliation Report
   ↓
7. Mark Period Reconciled (with user confirmation)
```

### A. Detailed Workflow Implementation
```python
def perform_reconciliation(bank_account_id, statement_date, bank_transactions):
    """
    Main reconciliation workflow
    """
    # Step 1: Get corresponding GL entries
    gl_entries = get_gl_entries_for_account(bank_account_id, statement_date)

    # Step 2: Auto-match high-confidence transactions
    auto_matched = []
    remaining_bank_txs = bank_transactions.copy()
    remaining_gl_entries = gl_entries.copy()

    for bank_tx in bank_transactions:
        match_result = advanced_matching(bank_tx, remaining_gl_entries)

        if match_result and match_result['confidence'] > 0.95:
            auto_matched.append({
                'bank_tx': bank_tx,
                'gl_entry': match_result['matched_entry'],
                'confidence': match_result['confidence']
            })

            # Remove matched items from consideration
            remaining_bank_txs.remove(bank_tx)
            if match_result.get('matched_entry'):
                remaining_gl_entries.remove(match_result['matched_entry'])
            elif match_result.get('matched_entries'):
                for entry in match_result['matched_entries']:
                    if entry in remaining_gl_entries:
                        remaining_gl_entries.remove(entry)

    # Step 3: Find suggested matches for review
    suggested_matches = []
    for bank_tx in remaining_bank_txs:
        match_result = advanced_matching(bank_tx, remaining_gl_entries)
        if match_result and 0.8 <= match_result['confidence'] <= 0.95:
            suggested_matches.append({
                'bank_tx_id': bank_tx.id,
                'gl_entry_id': match_result['matched_entry'].id if match_result.get('matched_entry') else None,
                'confidence': match_result['confidence'],
                'match_reason': match_result['match_reason']
            })

    # Step 4: Identify unmatched items requiring attention
    unmatched_items = {
        'bank_transactions': remaining_bank_txs,
        'gl_entries': remaining_gl_entries
    }

    # Step 5: Generate recommendations
    recommendations = generate_recommendations(unmatched_items)

    return {
        'auto_matched_count': len(auto_matched),
        'suggested_matches': suggested_matches,
        'unmatched_bank': len(remaining_bank_txs),
        'unmatched_gl': len(remaining_gl_entries),
        'recommendations': recommendations,
        'auto_matched': auto_matched
    }
```

## 7. API Integration

### Endpoint: POST /api/v1/ai/reconcile

#### Request Body
```json
{
  "bank_account_id": "bank_123",
  "statement_date": "2024-01-31",
  "bank_transactions": [
    {
      "id": "btx_001",
      "date": "2024-01-15",
      "amount": -1250.00,
      "description": "Check #1234 - Office Supplies",
      "transaction_type": "debit"
    },
    {
      "id": "btx_002",
      "date": "2024-01-20",
      "amount": 5000.00,
      "description": "ACH Deposit - Customer ABC",
      "transaction_type": "credit"
    }
  ],
  "start_date": "2024-01-01",
  "end_date": "2024-01-31",
  "user_id": "user_456"
}
```

#### Response Schema
```json
{
  "reconciliation_id": "rec_789",
  "bank_account_id": "bank_123",
  "statement_date": "2024-01-31",
  "auto_matched": 45,
  "suggested_matches": [
    {
      "bank_tx_id": "btx_1",
      "gl_entry_id": "je_456",
      "confidence": 0.88,
      "match_reason": "Amount exact, date within 2 days",
      "match_type": "fuzzy_date"
    }
  ],
  "unmatched_bank": 3,
  "unmatched_gl": 2,
  "recommendations": [
    {
      "type": "bank_fee",
      "transaction_id": "btx_999",
      "suggested_action": "create_journal_entry",
      "journal_entry": {
        "description": "Monthly service charge",
        "lines": [
          {
            "account": "5200",
            "debit": 25.00,
            "credit": 0
          },
          {
            "account": "1000",
            "debit": 0,
            "credit": 25.00
          }
        ]
      },
      "confidence": 0.9
    }
  ],
  "summary": {
    "beginning_balance": 25000.00,
    "ending_balance": 28500.00,
    "bank_total": 28500.00,
    "gl_total": 28475.00,
    "variance": 25.00,
    "status": "needs_review"
  },
  "confidence_score": 0.92,
  "next_steps": [
    "Review suggested matches",
    "Investigate unmatched items",
    "Create required journal entries"
  ]
}
```

## 8. Advanced Features

### A. Variance Analysis
```python
def analyze_variance(bank_balance, gl_balance, tolerance_percent=0.01):
    """
    Analyzes the variance between bank and GL balances
    """
    variance = abs(bank_balance - gl_balance)
    tolerance = max(abs(bank_balance), abs(gl_balance)) * tolerance_percent

    if variance <= tolerance:
        status = "balanced"
    elif variance <= tolerance * 5:
        status = "minor_variance"
    else:
        status = "significant_variance"

    return {
        'variance_amount': variance,
        'variance_percentage': (variance / max(abs(bank_balance), abs(gl_balance))) * 100,
        'tolerance': tolerance,
        'status': status,
        'recommended_action': get_variance_recommendation(status)
    }
```

### B. Pattern Recognition
```python
def detect_reconciliation_patterns(account_history):
    """
    Identifies patterns in historical reconciliation data
    """
    patterns = {}

    # Identify recurring mismatches
    recurring_mismatches = find_recurring_mismatches(account_history)
    patterns['recurring_mismatches'] = recurring_mismatches

    # Identify timing patterns
    timing_patterns = analyze_timing_patterns(account_history)
    patterns['timing_patterns'] = timing_patterns

    # Identify seasonal variations
    seasonal_variations = detect_seasonal_variations(account_history)
    patterns['seasonal_variations'] = seasonal_variations

    return patterns
```

## 9. Quality Assurance

### A. Match Validation
```python
def validate_match(bank_tx, gl_entry):
    """
    Validates that a proposed match makes accounting sense
    """
    validation_results = {
        'amount_match': abs(bank_tx.amount - gl_entry.amount) < 0.01,
        'sign_match': (bank_tx.amount > 0) == (gl_entry.amount > 0),
        'account_type_consistency': check_account_type_consistency(bank_tx, gl_entry),
        'timing_reasonableness': check_timing_reasonableness(bank_tx, gl_entry)
    }

    overall_valid = all(validation_results.values())

    return {
        'valid': overall_valid,
        'results': validation_results,
        'issues': [key for key, value in validation_results.items() if not value]
    }
```

### B. Confidence Adjustment Based on Validation
```python
def adjust_confidence_for_validation(match_result, validation_result):
    """
    Adjusts confidence based on validation results
    """
    if not validation_result['valid']:
        # Reduce confidence for invalid matches
        adjusted_confidence = match_result['confidence'] * 0.5
        return max(adjusted_confidence, 0.1)  # Don't go below 10%

    return match_result['confidence']
```

## 10. Performance Requirements

### A. Processing Speed
- **Small accounts** (< 100 transactions): < 5 seconds
- **Medium accounts** (100-500 transactions): < 15 seconds
- **Large accounts** (> 500 transactions): < 30 seconds

### B. Accuracy Standards
- **Exact matches**: 99.9% accuracy
- **High-confidence suggestions**: 95% accuracy
- **Overall reconciliation assistance**: 90% accuracy

### C. Scalability
- Support 1000+ concurrent reconciliations
- Horizontal scaling for peak periods
- Efficient database queries for large datasets