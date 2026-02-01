# CHART OF ACCOUNTS SPECIFICATION - AccountingAI Pro

**Specification Version:** 1.0
**Effective Date:** January 31, 2026
**Classification:** Foundation Specification

---

## OVERVIEW

This specification defines the structure, rules, and behavior for the Chart of Accounts (COA) in the AccountingAI Pro system. The COA serves as the foundation for all financial transactions and reporting, organizing accounts in a hierarchical structure that supports both operational efficiency and regulatory compliance.

---

## 1. DATA MODEL

### 1.1 SQL Schema Definition
```sql
CREATE TABLE chart_of_accounts (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL,
  code VARCHAR(20) UNIQUE NOT NULL,  -- e.g., "1000", "1010"
  name VARCHAR(255) NOT NULL,
  account_type ENUM('asset', 'liability', 'equity', 'revenue', 'expense') NOT NULL,
  account_subtype VARCHAR(50),  -- 'current_asset', 'fixed_asset', etc.
  parent_id UUID REFERENCES chart_of_accounts(id),  -- for hierarchy
  is_header BOOLEAN DEFAULT FALSE,  -- true = parent, false = leaf
  normal_balance ENUM('debit', 'credit') NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  is_system BOOLEAN DEFAULT FALSE,  -- cannot be deleted
  currency_code VARCHAR(3) DEFAULT 'USD',
  tax_applicable BOOLEAN DEFAULT FALSE,
  requires_dimension BOOLEAN DEFAULT FALSE,  -- job, department, etc.
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  created_by UUID,

  CONSTRAINT check_header_no_parent CHECK (
    NOT (is_header = TRUE AND parent_id IS NOT NULL)
  ),
  CONSTRAINT check_leaf_has_parent CHECK (
    is_header = TRUE OR parent_id IS NOT NULL
  )
);

CREATE INDEX idx_coa_tenant ON chart_of_accounts(tenant_id);
CREATE INDEX idx_coa_type ON chart_of_accounts(account_type);
CREATE INDEX idx_coa_parent ON chart_of_accounts(parent_id);
```

### 1.2 Field Descriptions
- **id**: Unique identifier for the account record
- **tenant_id**: Identifies which tenant owns this account
- **code**: Hierarchical account code (e.g., "1000", "1010")
- **name**: Descriptive name for the account
- **account_type**: Primary account classification
- **account_subtype**: Secondary classification for detailed reporting
- **parent_id**: Reference to parent account for hierarchical structure
- **is_header**: Indicates if account is a parent (header) or child (leaf)
- **normal_balance**: Expected balance direction (debit or credit)
- **is_active**: Controls whether the account can be used for transactions
- **is_system**: Prevents deletion of essential system accounts
- **currency_code**: Currency used for this account (multi-currency support)
- **tax_applicable**: Whether this account is subject to tax calculations
- **requires_dimension**: Whether this account requires additional dimensions (job, department, etc.)

---

## 2. ACCOUNT STRUCTURE

### 2.1 Standard Numbering Convention

#### Assets (1000-1999)
- **1000-1099**: Current Assets
- **1100-1199**: Cash & Equivalents
- **1200-1299**: Accounts Receivable
- **1300-1399**: Inventory
- **1400-1499**: Prepaid Expenses
- **1500-1999**: Fixed Assets

#### Liabilities (2000-2999)
- **2000-2099**: Current Liabilities
- **2100-2199**: Accounts Payable
- **2200-2299**: Accrued Expenses
- **2300-2999**: Long-term Liabilities

#### Equity (3000-3999)
- **3000-3099**: Common Stock
- **3100-3199**: Retained Earnings
- **3200-3999**: Additional Equity Accounts

#### Revenue (4000-4999)
- **4000-4099**: Sales Revenue
- **4100-4199**: Service Revenue
- **4200-4999**: Other Revenue

#### Expenses (5000-9999)
- **5000-5999**: Cost of Goods Sold
- **6000-6999**: Operating Expenses
- **7000-7999**: Administrative Expenses
- **8000-8999**: Selling Expenses
- **9000-9999**: Other Expenses

### 2.2 Hierarchy Example
```
1000 - Assets (HEADER)
├── 1100 - Current Assets (HEADER)
│   ├── 1110 - Cash in Bank (LEAF)
│   ├── 1120 - Petty Cash (LEAF)
│   └── 1130 - Accounts Receivable (LEAF)
└── 1500 - Fixed Assets (HEADER)
    ├── 1510 - Equipment (LEAF)
    └── 1520 - Accumulated Depreciation (LEAF)
```

### 2.3 Account Code Format
- Numeric format: 4 digits (e.g., 1000, 1010, 1015)
- Hierarchical: First digit indicates account type, subsequent digits indicate subcategories
- Flexible: Can extend to include decimals for deeper nesting (e.g., 1000.01)

---

## 3. BUSINESS RULES

### 3.1 Transaction Rules
- **LEAF Account Transactions**: Only leaf accounts (non-header) can have direct transactions posted to them
- **HEADER Account Balances**: Header accounts show rollup balances of all child accounts
- **Account Code Uniqueness**: Account codes must be unique per tenant
- **Normal Balance Rules**:
  - Assets and Expenses have DEBIT normal balance
  - Liabilities, Equity, and Revenue have CREDIT normal balance

### 3.2 Modification Rules
- **Transaction History Protection**: Cannot delete accounts with transaction history (archive instead)
- **Account Type Locking**: Cannot change account type once transactions exist
- **Circular Reference Prevention**: Cannot create circular hierarchies
- **Parent-Child Validation**: Parents must be header accounts, children must be leaf accounts

### 3.3 Validation Constraints
- **Header-Parent Rule**: Header accounts cannot have parents
- **Leaf-Parent Rule**: Leaf accounts must have parents (except root level)
- **Active Status**: Inactive accounts cannot be used for new transactions
- **System Account Protection**: System accounts cannot be deleted or modified

### 3.4 Business Logic Implementation
```typescript
interface ChartOfAccountsValidator {
  validateAccountCreation(account: Account): ValidationResult;
  validateAccountModification(account: Account, existing: Account): ValidationResult;
  validateHierarchy(account: Account): ValidationResult;
  validateTransaction(accountId: UUID, amount: Decimal, type: 'debit' | 'credit'): ValidationResult;
}

class ChartOfAccountsValidatorImpl implements ChartOfAccountsValidator {
  async validateAccountCreation(account: Account): Promise<ValidationResult> {
    // Check for duplicate code within tenant
    const existing = await this.findAccountByCode(account.tenant_id, account.code);
    if (existing) {
      return {
        valid: false,
        errors: [`Account code ${account.code} already exists for this tenant`]
      };
    }

    // Validate parent relationship
    if (account.parent_id) {
      const parent = await this.getAccountById(account.parent_id);
      if (!parent.is_header) {
        return {
          valid: false,
          errors: [`Parent account ${parent.code} must be a header account`]
        };
      }
    }

    // Validate normal balance based on account type
    if (!this.validateNormalBalance(account.account_type, account.normal_balance)) {
      return {
        valid: false,
        errors: [`Invalid normal balance for account type ${account.account_type}`]
      };
    }

    return { valid: true, errors: [] };
  }

  async validateAccountModification(account: Account, existing: Account): Promise<ValidationResult> {
    // Cannot change account type if transactions exist
    if (account.account_type !== existing.account_type) {
      const hasTransactions = await this.hasTransactionHistory(existing.id);
      if (hasTransactions) {
        return {
          valid: false,
          errors: ['Cannot change account type once transactions exist']
        };
      }
    }

    // Validate no circular reference
    if (account.parent_id && await this.wouldCreateCircularReference(account.id, account.parent_id)) {
      return {
        valid: false,
        errors: ['Cannot create circular hierarchy reference']
      };
    }

    return { valid: true, errors: [] };
  }

  private validateNormalBalance(accountType: string, normalBalance: string): boolean {
    const debitTypes = ['asset', 'expense'];
    const creditTypes = ['liability', 'equity', 'revenue'];

    if (debitTypes.includes(accountType)) {
      return normalBalance === 'debit';
    } else if (creditTypes.includes(accountType)) {
      return normalBalance === 'credit';
    }
    return false;
  }
}
```

---

## 4. TEMPLATES

### 4.1 Pre-configured Templates
The system provides pre-configured chart of accounts templates for different business types:

#### Small Business Template
- Simplified structure with essential accounts
- Minimal hierarchy depth
- Basic asset, liability, equity, revenue, and expense categories

#### E-commerce Template
- Additional inventory accounts
- Online payment processing accounts
- Shipping and fulfillment cost accounts
- Digital marketing expense categories

#### Professional Services Template
- Service revenue tracking
- Professional development expenses
- Client deposit accounts
- Project-based cost tracking

#### Manufacturing Template
- Raw materials inventory
- Work-in-progress accounts
- Finished goods inventory
- Manufacturing overhead categories
- Cost of goods sold breakdown

#### Non-profit Template
- Grant revenue accounts
- Program expense categories
- Fund accounting support
- Contribution tracking
- Restricted vs unrestricted funds

### 4.2 Template Customization
- Users can select a template and customize it after initial setup
- System preserves template structure while allowing additions
- Customizations are tracked separately from template accounts

---

## 5. AI SUGGESTIONS

### 5.1 Classification Agent Integration
The Classification Agent uses the chart of accounts to suggest appropriate accounts for transactions:

```typescript
interface AccountClassifier {
  suggestAccount(transactionDescription: string, amount: Decimal): AccountSuggestion[];
  learnFromUserMapping(transactionId: UUID, accountId: UUID): void;
  getConfidenceScore(suggestion: AccountSuggestion): number;
}

class AccountClassifierImpl implements AccountClassifier {
  async suggestAccount(transactionDescription: string, amount: Decimal): Promise<AccountSuggestion[]> {
    // Analyze transaction description
    const keywords = this.extractKeywords(transactionDescription);

    // Find matching accounts based on keywords and patterns
    const potentialMatches = await this.findMatchingAccounts(keywords);

    // Calculate confidence scores
    const suggestions = potentialMatches.map(account => ({
      account,
      confidence: this.calculateConfidence(account, keywords, amount),
      reason: this.generateReason(account, transactionDescription)
    }));

    // Sort by confidence score
    return suggestions.sort((a, b) => b.confidence - a.confidence);
  }

  learnFromUserMapping(transactionId: UUID, accountId: UUID): void {
    // Store user's mapping decision for future learning
    this.learningEngine.recordUserChoice(transactionId, accountId);
  }

  private calculateConfidence(account: Account, keywords: string[], amount: Decimal): number {
    // Weighted scoring based on:
    // - Keyword match relevance
    // - Historical usage patterns
    // - Amount range appropriateness
    // - Account type appropriateness

    let score = 0;

    // Keyword relevance
    for (const keyword of keywords) {
      if (account.name.toLowerCase().includes(keyword.toLowerCase())) {
        score += 30;
      }
      if (account.description?.toLowerCase().includes(keyword.toLowerCase())) {
        score += 20;
      }
    }

    // Historical usage (weight: 40%)
    const historicalScore = this.getHistoricalUsage(account.id, keywords);
    score += historicalScore * 0.4;

    // Amount range appropriateness (weight: 20%)
    const amountScore = this.getAmountRangeAppropriateness(account.id, amount);
    score += amountScore * 0.2;

    // Cap at 100
    return Math.min(score, 100);
  }
}
```

### 5.2 Machine Learning Integration
- The system learns from user's past account selections
- Confidence scores help users make informed decisions
- Suggestions improve over time based on user feedback

---

## 6. API ENDPOINTS

### 6.1 REST API Specification

#### GET /api/v1/chart-of-accounts
Retrieve chart of accounts for a tenant
```http
GET /api/v1/chart-of-accounts?tenant_id={id}&include_inactive={true|false}
Authorization: Bearer {token}
```

**Query Parameters:**
- `tenant_id`: Required - Tenant identifier
- `include_inactive`: Optional - Include inactive accounts (default: false)
- `account_type`: Optional - Filter by account type
- `parent_id`: Optional - Filter by parent account

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "code": "1000",
      "name": "Assets",
      "account_type": "asset",
      "account_subtype": "current_asset",
      "parent_id": null,
      "is_header": true,
      "normal_balance": "debit",
      "is_active": true,
      "is_system": true,
      "currency_code": "USD",
      "tax_applicable": false,
      "requires_dimension": false,
      "children": [
        {
          "id": "uuid",
          "code": "1100",
          "name": "Current Assets",
          "account_type": "asset",
          "is_header": true,
          "children": [
            {
              "id": "uuid",
              "code": "1110",
              "name": "Cash in Bank",
              "account_type": "asset",
              "is_header": false,
              "normal_balance": "debit",
              "is_active": true
            }
          ]
        }
      ]
    }
  ],
  "meta": {
    "total": 25,
    "filtered_total": 25
  }
}
```

#### POST /api/v1/chart-of-accounts
Create a new account
```http
POST /api/v1/chart-of-accounts
Authorization: Bearer {token}
Content-Type: application/json

{
  "tenant_id": "uuid",
  "code": "1115",
  "name": "Petty Cash",
  "account_type": "asset",
  "account_subtype": "cash_equivalent",
  "parent_id": "uuid",
  "normal_balance": "debit",
  "tax_applicable": false,
  "requires_dimension": false
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "code": "1115",
    "name": "Petty Cash",
    "account_type": "asset",
    "account_subtype": "cash_equivalent",
    "parent_id": "uuid",
    "is_header": false,
    "normal_balance": "debit",
    "is_active": true,
    "is_system": false,
    "currency_code": "USD",
    "tax_applicable": false,
    "requires_dimension": false,
    "created_at": "2026-01-31T12:00:00Z",
    "updated_at": "2026-01-31T12:00:00Z",
    "created_by": "uuid"
  }
}
```

#### PUT /api/v1/chart-of-accounts/{id}
Update an existing account
```http
PUT /api/v1/chart-of-accounts/{id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "Updated Account Name",
  "is_active": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "code": "1115",
    "name": "Updated Account Name",
    "account_type": "asset",
    "is_active": true,
    "updated_at": "2026-01-31T12:00:00Z"
  }
}
```

#### DELETE /api/v1/chart-of-accounts/{id}
Soft delete an account (archives instead of removing)
```http
DELETE /api/v1/chart-of-accounts/{id}
Authorization: Bearer {token}
```

**Response:**
```json
{
  "success": true,
  "message": "Account archived successfully"
}
```

#### GET /api/v1/chart-of-accounts/templates
Retrieve available chart of accounts templates
```http
GET /api/v1/chart-of-accounts/templates
Authorization: Bearer {token}
```

**Response:**
```json
{
  "data": [
    {
      "id": "small_business",
      "name": "Small Business",
      "description": "Basic chart of accounts for small businesses",
      "account_count": 25,
      "categories": ["assets", "liabilities", "equity", "revenue", "expenses"]
    },
    {
      "id": "ecommerce",
      "name": "E-commerce",
      "description": "Chart of accounts for online retail businesses",
      "account_count": 45,
      "categories": ["assets", "liabilities", "equity", "revenue", "expenses", "inventory"]
    }
  ]
}
```

### 6.2 API Validation Rules
- All endpoints validate tenant ownership
- Account creation validates uniqueness and hierarchy
- Account updates validate transaction history before type changes
- Soft deletes prevent data loss while maintaining referential integrity

---

## 7. VALIDATION RULES

### 7.1 Input Validation
- **Account Code**: Alphanumeric, no spaces, length 1-20 characters
- **Name**: 1-255 characters, no leading/trailing whitespace
- **Parent Relationship**: Parent must be a header account
- **Circular Reference**: Prevent creation of circular hierarchy

### 7.2 Business Logic Validation
- **Transaction History Check**: Prevent changes to accounts with transaction history
- **Account Type Consistency**: Maintain normal balance rules
- **Tenant Isolation**: Accounts belong to specific tenants only
- **System Account Protection**: Prevent modification of system accounts

### 7.3 Validation Implementation
```typescript
class ChartOfAccountsInputValidator {
  validateAccountCode(code: string): ValidationResult {
    if (!code || code.trim().length === 0) {
      return { valid: false, error: 'Account code is required' };
    }

    if (code.length > 20) {
      return { valid: false, error: 'Account code must be 20 characters or less' };
    }

    if (/[^\w]/.test(code)) {  // No spaces or special characters
      return { valid: false, error: 'Account code can only contain letters, numbers, and underscores' };
    }

    return { valid: true };
  }

  validateAccountName(name: string): ValidationResult {
    if (!name || name.trim().length === 0) {
      return { valid: false, error: 'Account name is required' };
    }

    if (name.length > 255) {
      return { valid: false, error: 'Account name must be 255 characters or less' };
    }

    if (name !== name.trim()) {
      return { valid: false, error: 'Account name cannot have leading or trailing whitespace' };
    }

    return { valid: true };
  }

  async validateParentRelationship(parentId: UUID | null, accountType: string): Promise<ValidationResult> {
    if (!parentId) {
      // Root level accounts are allowed
      return { valid: true };
    }

    const parent = await this.getAccountById(parentId);
    if (!parent) {
      return { valid: false, error: 'Parent account does not exist' };
    }

    if (!parent.is_header) {
      return { valid: false, error: 'Parent account must be a header account' };
    }

    return { valid: true };
  }
}
```

---

**Document Classification:** Foundation Specification
**Review Cycle:** Annual review required or upon accounting standard changes
**Approval Authority:** Chief Financial Officer and Product Owner