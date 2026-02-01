# Bank Account Management Specification

## Overview
This specification defines the bank account management system for the accounting platform. It encompasses bank account setup, transaction importing, reconciliation, and automated categorization features.

## Data Model

### Bank Accounts Table
```sql
CREATE TABLE banking.bank_accounts (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  account_name VARCHAR(255),
  account_number VARCHAR(50),
  bank_name VARCHAR(255),
  currency CHAR(3) DEFAULT 'USD',
  account_type ENUM('checking', 'savings', 'credit_card'),
  gl_account_id UUID NOT NULL, -- Link to Chart of Accounts
  opening_balance DECIMAL(15,2),
  current_balance DECIMAL(15,2),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP,
  updated_at TIMESTAMP,

  FOREIGN KEY (tenant_id) REFERENCES tenants.id,
  FOREIGN KEY (gl_account_id) REFERENCES chart_of_accounts.id
);
```

### Bank Transactions Table
```sql
CREATE TABLE banking.bank_transactions (
  id UUID PRIMARY KEY,
  bank_account_id UUID NOT NULL,
  transaction_date DATE,
  description TEXT,
  amount DECIMAL(15,2),
  transaction_type ENUM('debit', 'credit'),
  balance DECIMAL(15,2),
  reference VARCHAR(100),
  is_reconciled BOOLEAN DEFAULT FALSE,
  matched_journal_id UUID,
  imported_at TIMESTAMP,
  created_at TIMESTAMP,

  FOREIGN KEY (bank_account_id) REFERENCES banking.bank_accounts.id,
  FOREIGN KEY (matched_journal_id) REFERENCES transactions.journal_entries.id
);
```

## Bank Feeds Integration

### Plaid API Integration
- OAuth-based connection to financial institutions
- Real-time transaction data synchronization
- Account balance monitoring
- Transaction categorization by Plaid
- Error handling and retry mechanisms

### Yodlee Support
- Alternative aggregation service for broader coverage
- Support for institutions not available through Plaid
- Backup data source for redundancy
- Consistent data format normalization

### Manual Import Options
- CSV file upload with predefined format
- OFX/QFX file import support
- Bulk transaction upload
- Data validation and error reporting

## Transaction Import Specifications

### CSV Import Format
```
Date,Description,Amount,Type,Reference,Balance
2024-01-15,AMAZON.COM,-45.99,debit,REF12345,1234.56
2024-01-16,SALARY DEPOSIT,2500.00,credit,REF67890,3734.56
```

### Supported File Formats
- CSV with header row
- OFX (Open Financial Exchange)
- QFX (Quicken Financial Exchange)
- JSON (for API imports)

### Import Validation Rules
- Date format validation (YYYY-MM-DD)
- Amount format validation (decimal with 2 places)
- Required field validation
- Duplicate transaction detection
- Balance consistency checks

## Auto-Categorization (AI)

### Machine Learning Approach
- Learn from historical categorization patterns
- Improve suggestions over time
- Handle variations in transaction descriptions
- Support for multiple languages

### AI Categorization Prompt
```
Analyze the following bank transaction and suggest the most appropriate GL account:
Transaction Description: "{transaction_description}"
Amount: {amount}
Transaction Type: {transaction_type}
Previous Category History: {category_history}

Consider the following factors:
- Keywords in the description
- Amount range and frequency
- Similar past transactions
- Business type and industry
- Account type (checking/savings/credit_card)

Return the suggested GL account code, description, and confidence score (0-100%).
```

### Vendor/Customer Matching
- Identify recurring vendor payments
- Match to existing vendor records
- Suggest customer names for deposit transactions
- Learn from user corrections

### Confidence Scoring
- High confidence (>90%): Auto-apply category
- Medium confidence (70-90%): Suggest for review
- Low confidence (<70%): Manual categorization required

## Bank Rules Engine

### Rule Configuration
- Define rules based on transaction description keywords
- Specify target GL accounts for automatic posting
- Set priority levels for rule conflicts
- Support for regular expressions

### Sample Rules
```
IF description CONTAINS "AMAZON"
THEN categorize as "Expense: Office Supplies"

IF description CONTAINS "STARBUCKS"
THEN categorize as "Expense: Meals & Entertainment"

IF description CONTAINS "PAYROLL"
THEN categorize as "Expense: Salaries"

IF description CONTAINS "DEPOSIT"
THEN categorize as "Income: Service Revenue"
```

### Auto-Journal Creation
- Automatically create journal entries for matched transactions
- Use contra-accounts based on transaction type
- Link journal entries to bank transactions
- Maintain audit trail

## Matching Algorithm

### Transaction Matching Logic
```
FOR each new bank transaction DO
  1. Check for existing exact matches (amount + date + description)
  2. Search for journal entries within date range (±3 days)
  3. Compare amounts for similarity (±$0.01 tolerance)
  4. Analyze description keywords for semantic similarity
  5. Apply bank rules for automatic matching
  6. Use AI suggestions for categorization
  7. Present potential matches to user for confirmation
END FOR
```

### Fuzzy Matching Criteria
- Amount matching (exact or within tolerance)
- Date proximity (within configurable days)
- Description similarity (using string distance algorithms)
- Vendor/customer name matching
- Reference number matching

### Reconciliation Workflow
1. Import new transactions
2. Apply matching algorithm
3. Auto-categorize with AI
4. Apply bank rules
5. Present unmatched transactions for manual review
6. Confirm matches and post journal entries
7. Mark transactions as reconciled

## Integration API Specifications

### Plaid Integration Endpoints
- `POST /api/banking/plaid/link-token` - Create link token for Plaid connection
- `POST /api/banking/plaid/exchange-token` - Exchange public token for access token
- `GET /api/banking/plaid/accounts` - Get connected accounts
- `GET /api/banking/plaid/transactions` - Fetch transactions for account
- `POST /api/banking/plaid/remove-item` - Disconnect financial institution

### Bank Account Management Endpoints
- `POST /api/banking/accounts` - Create bank account
- `GET /api/banking/accounts` - List bank accounts
- `GET /api/banking/accounts/{id}` - Get specific account
- `PUT /api/banking/accounts/{id}` - Update account
- `DELETE /api/banking/accounts/{id}` - Delete account
- `POST /api/banking/accounts/{id}/sync` - Sync transactions from bank

### Transaction Management Endpoints
- `GET /api/banking/accounts/{id}/transactions` - Get account transactions
- `POST /api/banking/accounts/{id}/transactions/import` - Import transactions
- `PUT /api/banking/transactions/{id}/categorize` - Update transaction category
- `POST /api/banking/transactions/{id}/match` - Match transaction to journal entry
- `POST /api/banking/transactions/reconcile` - Reconcile transactions

### Rules Management Endpoints
- `POST /api/banking/rules` - Create bank rule
- `GET /api/banking/rules` - List bank rules
- `PUT /api/banking/rules/{id}` - Update bank rule
- `DELETE /api/banking/rules/{id}` - Delete bank rule

## Security & Access Control

- Encrypted storage of bank credentials
- PCI DSS compliance for card data
- OAuth token management
- Role-based access to bank accounts
- Audit logging for all bank operations
- Secure API communication with financial institutions
- Regular security assessments of third-party integrations

## Error Handling & Monitoring

### Retry Mechanisms
- Failed bank sync retries with exponential backoff
- Queue management for processing failures
- Alert system for persistent failures
- Manual intervention workflows

### Monitoring Metrics
- Sync success/failure rates
- Average transaction processing time
- Categorization accuracy metrics
- User acceptance rates for AI suggestions
- Third-party API health status

## Performance Considerations

- Efficient bulk import for large transaction volumes
- Asynchronous processing for AI categorization
- Caching of frequently accessed data
- Database indexing for transaction searches
- Pagination for large transaction lists