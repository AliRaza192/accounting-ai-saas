# Chart of Accounts Specification

## Overview
The Chart of Accounts (COA) is a foundational component of the accounting system that organizes financial accounts into a hierarchical structure. This specification defines the data model, business rules, and API endpoints for managing accounts in a multi-tenant environment.

## Data Model

### Table Definition
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

## Account Structure

### Standard Numbering System
Accounts follow a standardized numbering convention to ensure consistency across organizations:

#### Asset Accounts (1000-1999)
- **1000-1099**: Current Assets
- **1100-1199**: Cash & Equivalents
- **1200-1299**: Accounts Receivable
- **1300-1399**: Inventory
- **1400-1499**: Prepaid Expenses
- **1500-1999**: Fixed Assets

#### Liability Accounts (2000-2999)
- **2000-2099**: Current Liabilities
- **2100-2199**: Accounts Payable
- **2200-2299**: Accrued Expenses
- **2300-2999**: Long-term Liabilities

#### Equity Accounts (3000-3999)
- **3000-3999**: Owner's Equity, Retained Earnings, etc.

#### Revenue Accounts (4000-4999)
- **4000-4999**: Sales Revenue, Service Revenue, Other Income

#### Expense Accounts (5000-9999)
- **5000-9999**: Operating Expenses, Cost of Goods Sold, etc.

### Hierarchy Example
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

## Business Rules

### Core Rules
1. **Transaction Restrictions**: Only LEAF accounts can have transactions posted to them
2. **Rollup Display**: HEADER accounts show aggregated balances from their children
3. **Uniqueness**: Account codes must be unique per tenant
4. **Deletion Policy**: Cannot delete accounts with transaction history (archive instead)
5. **Account Type Immutability**: Cannot change account type once transactions exist

### Balance Rules
- **DEBIT Normal Balance**: Assets and Expenses
- **CREDIT Normal Balance**: Liabilities, Equity, and Revenue

### Hierarchy Constraints
- Parent accounts must be HEADER type
- Child accounts must be LEAF type
- No circular references allowed in the hierarchy
- Maximum nesting depth limited to 5 levels

## Templates

### Pre-configured Templates
The system provides industry-specific templates:
- Small Business
- E-commerce
- Professional Services
- Manufacturing
- Non-profit

Templates can be customized after selection but core system accounts remain protected.

## AI Integration

### Classification Agent
- Suggests appropriate accounts based on transaction descriptions
- Learns from user's historical account mappings
- Provides confidence scores for suggestions
- Continuously improves recommendations

## API Endpoints

### REST API
```
GET    /api/v1/chart-of-accounts?tenant_id={id}&include_inactive={bool}&parent_id={id}
POST   /api/v1/chart-of-accounts
PUT    /api/v1/chart-of-accounts/{id}
DELETE /api/v1/chart-of-accounts/{id}  -- soft delete only
GET    /api/v1/chart-of-accounts/templates
GET    /api/v1/chart-of-accounts/templates/{industry}
POST   /api/v1/chart-of-accounts/templates/{industry}/apply
GET    /api/v1/chart-of-accounts/{id}/hierarchy
GET    /api/v1/chart-of-accounts/search?q={query}&tenant_id={id}
```

### Request/Response Examples

#### Create Account
```json
{
  "tenant_id": "uuid",
  "code": "1110",
  "name": "Cash in Bank",
  "account_type": "asset",
  "account_subtype": "current_asset",
  "parent_id": "uuid", // Optional for headers
  "is_header": false,
  "normal_balance": "debit",
  "currency_code": "USD",
  "tax_applicable": false,
  "requires_dimension": false
}
```

#### Response
```json
{
  "id": "uuid",
  "tenant_id": "uuid",
  "code": "1110",
  "name": "Cash in Bank",
  "account_type": "asset",
  "account_subtype": "current_asset",
  "parent_id": "uuid",
  "is_header": false,
  "normal_balance": "debit",
  "is_active": true,
  "is_system": false,
  "currency_code": "USD",
  "tax_applicable": false,
  "requires_dimension": false,
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid"
}
```

## Validation Rules

### Input Validation
- Account code: Alphanumeric, max 20 characters, no spaces
- Account name: 1-255 characters, no leading/trailing whitespace
- Parent relationship: Parent must be a HEADER account
- Hierarchy: Prevent circular references
- Uniqueness: Code must be unique per tenant

### Business Logic Validation
- Cannot create transactions on HEADER accounts
- Cannot delete accounts with transaction history
- Cannot modify account type with existing transactions
- Currency must be valid ISO 4217 code
- Normal balance must match account type rules

## Audit Trail
All changes to the chart of accounts are tracked with:
- Creation/modification timestamps
- User who made changes
- Previous values for updates
- Reason for changes (optional field)

## Security Considerations
- Multi-tenant isolation enforced at database level
- System accounts cannot be modified/deleted
- Role-based access control for account creation/modification
- Soft deletes prevent permanent data loss