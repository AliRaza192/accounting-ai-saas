# AI Classification Agent Specification

## 1. Agent Purpose

The AI Classification Agent is responsible for:
- Automatically categorizing financial transactions
- Suggesting appropriate GL accounts for transactions
- Learning from user corrections to improve accuracy
- Matching vendors/customers to existing records

## 2. Classification Model

### Input Parameters
- Transaction description
- Transaction amount
- Vendor information
- Additional context (date, category hints)

### Output Structure
- Suggested GL account
- Confidence score (0.0 to 1.0)
- Reasoning explanation
- Alternative suggestions

### Training Source
- Historical user mappings
- Organizational transaction patterns
- Industry-standard classifications

## 3. Learning Algorithm

```python
def classify_transaction(description, amount, vendor):
    """
    Main classification algorithm that attempts multiple matching strategies
    in order of reliability, returning the first successful match with confidence.
    """
    # 1. Exact match from history
    if exact_match_exists(description):
        return historical_account, confidence=1.0

    # 2. Fuzzy match (Levenshtein distance)
    similar = find_similar_descriptions(description, threshold=0.8)
    if similar:
        return similar.account, confidence=0.9

    # 3. Keyword matching
    keywords = extract_keywords(description)
    matched_accounts = keyword_to_account_mapping(keywords)
    if matched_accounts:
        return most_common(matched_accounts), confidence=0.7

    # 4. AI prediction (Claude API)
    return ask_claude_for_classification(description), confidence=0.6
```

### Matching Priority Order
1. **Historical Matches** (Exact) - Highest confidence
2. **Fuzzy Matches** - High confidence
3. **Keyword Matches** - Medium confidence
4. **AI Prediction** - Lower confidence

## 4. Classification Rules

### Predefined Mappings
| Keyword Pattern | Suggested Account | Default Category |
|----------------|-------------------|------------------|
| "AMAZON" | Office Supplies (Expense) | Operating Expense |
| "SALARY" | Payroll Expense | Labor Cost |
| "RENT" | Rent Expense | Operating Expense |
| "INVOICE #" | Accounts Receivable (Asset) | Asset |
| "PAYROLL" | Payroll Taxes | Liability |
| "INSURANCE" | Insurance Expense | Operating Expense |
| "TRAVEL" | Travel Expense | Operating Expense |
| "MEALS" | Meals & Entertainment | Operating Expense |

### Rule Application Priority
1. Vendor-specific rules take precedence
2. Description patterns applied second
3. Amount-based heuristics as fallback

## 5. Confidence Thresholds

### Automatic Processing
- **> 95% Confidence**: Auto-apply classification without review
- **80-95% Confidence**: Suggest classification, require user approval
- **< 80% Confidence**: Flag for manual review and human decision

### Confidence Calculation Factors
- Match certainty (exact vs fuzzy)
- Historical accuracy of similar classifications
- Transaction amount unusualness
- Vendor classification consistency

## 6. Continuous Learning System

### Learning Triggers
- When user accepts a suggested classification
- When user corrects an auto-applied classification
- When user manually selects an alternative suggestion
- Weekly batch learning from approved classifications

### Learning Storage
- Store corrected pairs: `(description, amount, vendor) → correct_account`
- Maintain confidence levels for learned mappings
- Track effectiveness of different matching strategies

### Model Retraining Schedule
- Daily incremental updates for immediate corrections
- Weekly full model retraining for comprehensive learning
- Monthly performance analysis and algorithm adjustments

## 7. Agent Boundaries

### CAN Do
- Suggest appropriate account mappings based on learned patterns
- Categorize transactions by type and business function
- Learn from user corrections and improve over time
- Provide confidence scores and reasoning explanations

### CANNOT Do
- Auto-apply classifications when confidence < 95%
- Create new accounts in the chart of accounts
- Modify existing chart of accounts structure
- Override user decisions or force classifications

## 8. API Integration

### Endpoint: POST /api/v1/ai/classify

#### Request Body
```json
{
  "description": "Amazon Web Services",
  "amount": 150.00,
  "vendor": "AWS",
  "transaction_date": "2024-01-15",
  "category_hint": "technology",
  "previous_classifications": [
    {
      "description": "Amazon Web Services",
      "account": "6500",
      "confidence": 0.95
    }
  ]
}
```

#### Response Schema
```json
{
  "suggested_account": {
    "id": "uuid",
    "code": "6500",
    "name": "Cloud Services Expense"
  },
  "confidence": 0.92,
  "reasoning": "Matched 15 similar transactions",
  "alternative_suggestions": [
    {
      "account": {
        "id": "uuid",
        "code": "6400",
        "name": "IT Services Expense"
      },
      "confidence": 0.78,
      "reasoning": "General IT services pattern"
    }
  ],
  "learning_applied": true,
  "match_type": "historical_exact"
}
```

#### Status Values
- `auto_applied`: Classification applied without review
- `suggested`: Suggestion awaiting approval
- `flagged`: Requires manual review
- `error`: Processing failed

## 9. Claude API Prompt for Classification

### Primary Classification Prompt
```
You are a financial expert specializing in General Ledger account classification. Based on the transaction details provided, suggest the most appropriate GL account from the company's chart of accounts.

Transaction Details:
- Description: {description}
- Amount: {amount}
- Vendor: {vendor}

Consider:
1. Business function of the expense/revenue
2. Standard accounting practices for this type of transaction
3. Similar historical classifications
4. Company's industry and business model

Return your response in this exact JSON format:
{
  "account_code": "string",
  "account_name": "string",
  "category": "string",
  "confidence": float,
  "reasoning": "brief explanation of your choice"
}

Account categories: Revenue, Cost of Goods Sold, Operating Expense, Other Income, Other Expense, Asset, Liability, Equity.
```

### Alternative Suggestions Prompt
```
Provide 2-3 alternative GL account suggestions for this transaction that might also be appropriate, with slightly lower confidence than your primary recommendation.

Transaction: {description} ({amount}) from {vendor}

Format: Array of objects with account_code, account_name, confidence, and reasoning.
```

## 10. Learning Algorithm Implementation

### Historical Match Detection
```python
def find_exact_matches(description, vendor=None):
    """Find exact matches in historical data"""
    matches = []
    for record in historical_transactions:
        if record.description.lower() == description.lower():
            if vendor and record.vendor.lower() != vendor.lower():
                continue
            matches.append(record)
    return matches
```

### Fuzzy Matching Algorithm
```python
def find_fuzzy_matches(description, threshold=0.8):
    """Find similar descriptions using Levenshtein distance"""
    import difflib

    matches = []
    for record in historical_transactions:
        similarity = difflib.SequenceMatcher(
            None,
            description.lower(),
            record.description.lower()
        ).ratio()

        if similarity >= threshold:
            matches.append({
                'record': record,
                'similarity': similarity
            })

    return sorted(matches, key=lambda x: x['similarity'], reverse=True)
```

### Keyword Extraction and Mapping
```python
def extract_keywords(description):
    """Extract relevant keywords for classification"""
    import re

    # Remove common stop words and extract meaningful terms
    stop_words = {'the', 'and', 'or', 'of', 'to', 'for', 'with', 'on', 'at'}
    words = re.findall(r'\b\w+\b', description.lower())
    return [word for word in words if word not in stop_words]
```

## 11. Keyword Mappings

### Technology Services
- "aws", "amazon web services", "cloud" → Cloud Services Expense (6500)
- "microsoft", "office", "software" → Software License Expense (6600)
- "google", "gcp", "google cloud" → Cloud Services Expense (6500)

### Office Expenses
- "amazon", "amazon.com" → Office Supplies (6100)
- "fedex", "ups", "shipping" → Shipping & Delivery (6200)
- "postage", "mail" → Postage & Delivery (6200)

### Utilities
- "electric", "power", "pg&e" → Utilities - Electric (6300)
- "gas", "natural gas" → Utilities - Gas (6310)
- "water" → Utilities - Water (6320)

### Professional Services
- "consulting", "advisory" → Professional Services (6400)
- "legal", "attorney" → Legal & Professional (6410)
- "accounting", "cpa" → Accounting Services (6420)

## 12. Test Cases with Edge Cases

### Test Case 1: High Confidence Exact Match
- **Input**: Description: "Amazon Web Services", Amount: 150.00, Vendor: "AWS"
- **Expected**: Cloud Services Expense (6500), Confidence: 1.0
- **Reason**: Exact match in historical data

### Test Case 2: Fuzzy Match with High Confidence
- **Input**: Description: "Amazon cloud services monthly", Amount: 145.50, Vendor: "AWS"
- **Expected**: Cloud Services Expense (6500), Confidence: 0.9
- **Reason**: High similarity to historical AWS transactions

### Test Case 3: Keyword-Based Classification
- **Input**: Description: "Office supplies from Amazon", Amount: 89.99, Vendor: "Amazon"
- **Expected**: Office Supplies (6100), Confidence: 0.7
- **Reason**: Contains "office" and "supplies" keywords

### Test Case 4: AI-Predicted Classification
- **Input**: Description: "Unusual quarterly maintenance fee", Amount: 1200.00, Vendor: "Unknown"
- **Expected**: Maintenance Expense (6700), Confidence: 0.6
- **Reason**: AI classification with no historical matches

### Test Case 5: Negative Amount (Credit)
- **Input**: Description: "Customer payment received", Amount: -500.00, Vendor: "Customer ABC"
- **Expected**: Accounts Receivable (1200), Confidence: 0.95
- **Reason**: Credit transaction likely represents payment

### Test Case 6: Ambiguous Description
- **Input**: Description: "Monthly charge", Amount: 99.99, Vendor: "Generic Service"
- **Expected**: Multiple suggestions flagged for review
- **Reason**: Insufficient information for confident classification

### Test Case 7: Large Unusual Amount
- **Input**: Description: "Equipment purchase", Amount: 25000.00, Vendor: "Office Depot"
- **Expected**: Equipment (Fixed Assets), Confidence: 0.85
- **Reason**: High amount suggests capital expenditure

### Test Case 8: International Vendor
- **Input**: Description: "Software license EU", Amount: 450.00, Vendor: "European Software Co."
- **Expected**: Software License Expense (6600), Confidence: 0.75
- **Reason**: Software keyword with international context

### Test Case 9: Seasonal Pattern
- **Input**: Description: "Holiday party catering", Amount: 2000.00, Vendor: "Catering Co", Date: "Dec 15"
- **Expected**: Meals & Entertainment (6800), Confidence: 0.85
- **Reason**: Seasonal pattern recognition for holiday expenses

### Test Case 10: Learning Application
- **Input**: Description: "Previously misclassified item", Amount: 75.00, Vendor: "Vendor XYZ"
- **Expected**: Previously corrected account, Confidence: 1.0
- **Reason**: Applied learning from user correction

## 13. Performance Metrics

- **Accuracy Rate**: Percentage of correct auto-classifications
- **Learning Effectiveness**: Improvement in accuracy over time
- **User Acceptance Rate**: Percentage of suggestions accepted
- **Processing Speed**: Average response time under 2 seconds