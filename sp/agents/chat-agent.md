# AI Chat Agent (Virtual Accountant) Specification

## 1. Agent Purpose

The AI Chat Agent serves as a virtual accountant that:
- Answers accounting queries in natural language
- Retrieves financial data through conversational interfaces
- Explains transactions and their classifications
- Generates ad-hoc reports based on user requests
- Guides users through accounting processes and workflows

## 2. Sample Conversations

### A. Data Retrieval

**User:** "What's my cash balance?"

**Agent:** "Your current cash balance is $42,350 across 3 accounts:

Main Checking: $35,000
Savings: $7,000
Petty Cash: $350"

**User:** "Show me expenses over $5,000 last month"

**Agent:** "Found 4 expenses over $5,000 in January:

Rent: $8,000 (Jan 1)
Marketing Agency: $12,500 (Jan 15)
Equipment Purchase: $7,200 (Jan 20)
Legal Fees: $6,000 (Jan 28)

Total: $33,700
Would you like a detailed breakdown?"

### B. Transaction Explanation

**User:** "Why is there a $500 charge from AWS?"

**Agent:** "The $500 AWS charge on Jan 15 was:

Classified as: Cloud Services Expense
Account: 6500
Attached invoice: INV-AWS-12345
Journal Entry: JE-00421

This was auto-categorized based on your historical AWS expenses.
Should I change the classification?"

### C. Process Guidance

**User:** "How do I record a customer payment?"

**Agent:** "To record a customer payment:

Go to Invoices → Find the invoice
Click 'Record Payment'
Enter:

Payment date
Amount received
Bank account (where deposited)
Payment method

System will auto-create journal entry:
Debit: Bank Account
Credit: Accounts Receivable

Would you like me to walk you through it with a specific invoice?"

### D. Report Generation

**User:** "Compare my revenue this quarter vs last quarter"

**Agent:** "Q1 2024 vs Q4 2023 Revenue Comparison:
Q1 2024: $150,000
Q4 2023: $135,000
Change: +$15,000 (+11%)
Breakdown:

Product Sales: +$10K
Service Revenue: +$5K
Other Income: No change

Top customers this quarter:

ABC Corp: $45K
XYZ Inc: $30K
DEF Ltd: $25K

[Download Full Report]"

## 3. Knowledge Base

### A. Accounting Principles
- GAAP and IFRS fundamentals
- Double-entry bookkeeping
- Revenue recognition principles
- Expense matching principles
- Asset/liability classification

### B. System Features
- Navigation guides for all modules
- Feature explanations and tutorials
- Best practices for system usage
- Integration capabilities
- Reporting capabilities

### C. Common Workflows
- Invoice creation and management
- Payment recording procedures
- Expense approval processes
- Month-end closing procedures
- Bank reconciliation steps

### D. Troubleshooting
- Common error messages and solutions
- Data entry corrections
- Permission issues resolution
- Integration problems
- Performance optimization tips

## 4. Multi-Turn Context Management

### A. Conversation History
- Maintains context for up to 50 exchanges
- Tracks entities mentioned in conversation
- Preserves user intent across turns
- Handles topic shifts gracefully

### B. Follow-up Questions
- Recognizes implicit references to previous topics
- Disambiguates pronouns and references
- Provides relevant context from earlier exchanges
- Offers clarifying questions when needed

### C. Clarification Handling
- Identifies ambiguous requests
- Asks targeted questions for clarification
- Confirms understanding before proceeding
- Provides options when multiple interpretations exist

## 5. Agent Boundaries

### CAN Do
- Answer accounting queries and provide explanations
- Retrieve data within user's permission scope
- Explain transactions and their classifications
- Generate reports based on user requests
- Guide users through accounting workflows and processes

### CANNOT Do
- Execute financial transactions directly via chat interface
- Share data across different tenant organizations
- Bypass role-based access control restrictions
- Modify or update financial data in the system

### Requires Approval
- Bulk data export requests (exceeding 1000 records)
- Access to sensitive financial information
- Sharing of confidential business data

## 6. Security Framework

### A. User Permission Validation
- Verifies user permissions before data retrieval
- Applies row-level security filters
- Respects organizational boundaries
- Enforces access control policies

### B. Multi-Tenant Isolation
- Strict data separation between tenants
- Tenant-specific data access controls
- Isolated conversation contexts
- Prevents cross-tenant data leakage

### C. Data Privacy Controls
- No personally identifiable information in logs
- Sanitized query logging for debugging
- Encrypted conversation storage
- Automatic data retention policies

### D. Audit Trail
- Logs all user interactions for compliance
- Tracks data access and queries
- Monitors for suspicious activities
- Maintains audit trails for accountability

## 7. API Integration

### Endpoint: POST /api/v1/ai/chat

#### Request Body
```json
{
  "message": "What's my cash balance?",
  "conversation_id": "conv_123",
  "user_id": "user_456",
  "user_context": {
    "role": "accountant",
    "permissions": ["read_financial_data", "generate_reports"],
    "organization_id": "org_789",
    "department": "finance"
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "client_info": {
    "platform": "web",
    "device": "desktop",
    "version": "1.0"
  }
}
```

#### Response Schema
```json
{
  "message": "Your current cash balance is $42,350...",
  "data": {
    "cash_accounts": [
      {
        "account_name": "Main Checking",
        "balance": 35000,
        "account_type": "checking"
      },
      {
        "account_name": "Savings",
        "balance": 7000,
        "account_type": "savings"
      },
      {
        "account_name": "Petty Cash",
        "balance": 350,
        "account_type": "petty_cash"
      }
    ]
  },
  "suggestions": [
    "Show me this month's expenses",
    "Generate cash flow report",
    "Compare to previous month"
  ],
  "requires_action": false,
  "context_preserved": true,
  "confidence": 0.95
}
```

#### Alternative Response (Requires Action)
```json
{
  "message": "I found multiple cash accounts. Which one would you like to see?",
  "data": {
    "account_options": [
      {"id": "acc_1", "name": "Main Checking", "balance": 35000},
      {"id": "acc_2", "name": "Business Savings", "balance": 15000}
    ]
  },
  "suggestions": [],
  "requires_action": true,
  "action_type": "selection",
  "context_preserved": true
}
```

## 8. Claude API System Prompt

```
You are a professional virtual accountant assistant. Your role is to help users with their accounting and financial questions using natural language.

Guidelines:
1. Provide accurate, professional accounting advice based on GAAP/IFRS principles
2. Only provide information from the user's own data - never make up numbers
3. If you don't have access to specific data, say so clearly
4. Explain accounting concepts in clear, business-appropriate language
5. When explaining transactions, provide context about accounts and classifications
6. Guide users through system workflows step-by-step
7. Respect user permissions - don't access data they can't see
8. Never execute transactions - only guide users to the right screens
9. Be helpful but don't bypass security controls
10. Ask clarifying questions when needed

Capabilities:
- Answer accounting questions
- Retrieve and explain financial data
- Explain transaction classifications
- Guide users through processes
- Generate reports based on requests

Limitations:
- Cannot execute transactions
- Cannot access data outside user permissions
- Cannot share data between organizations
- Cannot modify financial records

Always prioritize accuracy and user safety over being helpful.
```

## 9. Conversation Flow Examples

### Flow 1: Multi-turn Data Inquiry
```
User: "What were my expenses last month?"
AI: "Last month (January) you had $25,000 in total expenses..."
User: "What about just office supplies?"
AI: "Office supplies expenses in January were $2,300..."
User: "Who did I pay?"
AI: "You paid Office Depot $1,200 and Staples $1,100..."
```

### Flow 2: Process Guidance with Context
```
User: "How do I record a refund?"
AI: "To record a customer refund, go to Invoices..."
User: "I meant vendor refund"
AI: "Apologies! For a vendor refund, go to Bills..."
User: "Okay, what account should I use?"
AI: "For vendor refunds, typically use Accounts Payable..."
```

### Flow 3: Report Generation Chain
```
User: "Show me revenue this quarter"
AI: "Q1 revenue was $150,000..."
User: "Compare to last quarter"
AI: "Q4 2023 was $135,000, so +11% growth..."
User: "Break down by customer"
AI: "Top customers: ABC Corp ($45K), XYZ Inc ($30K)..."
```

## 10. Knowledge Base Structure

### A. Hierarchical Organization
```
Accounting Knowledge Base
├── Fundamentals
│   ├── GAAP Principles
│   ├── IFRS Standards
│   └── Bookkeeping Basics
├── System Features
│   ├── Navigation Guides
│   ├── Feature Tutorials
│   └── Best Practices
├── Workflows
│   ├── Invoice Management
│   ├── Payment Processing
│   └── Month-End Close
└── Troubleshooting
    ├── Error Messages
    ├── Common Issues
    └── Solutions
```

### B. Dynamic Content Updates
- Regular updates from accounting standards bodies
- New system features and enhancements
- Common user questions and answers
- Industry-specific guidelines

### C. Personalization
- Organization-specific chart of accounts
- User role-based content filtering
- Department-specific procedures
- Custom workflow adaptations

## 11. Security Controls

### A. Access Control Implementation
```python
def validate_user_access(user_id, requested_data):
    """
    Validates that user has permission to access requested data
    """
    user_permissions = get_user_permissions(user_id)

    if not has_read_permission(user_permissions, requested_data.type):
        raise UnauthorizedAccessError("User lacks permission to access this data")

    # Apply row-level security
    filtered_data = apply_security_filters(requested_data, user_permissions)

    return filtered_data
```

### B. Data Sanitization
```python
def sanitize_response(response_data):
    """
    Removes sensitive information from responses
    """
    sanitized = response_data.copy()

    # Remove PII from logs
    if 'customer_details' in sanitized:
        sanitized['customer_details'] = mask_sensitive_fields(
            sanitized['customer_details']
        )

    # Log sanitized version only
    log_interaction(sanitized)

    return sanitized
```

### C. Tenant Isolation
```python
def enforce_tenant_isolation(user_org_id, requested_org_id):
    """
    Ensures user can only access their organization's data
    """
    if user_org_id != requested_org_id:
        raise CrossTenantAccessError(
            f"User from org {user_org_id} cannot access org {requested_org_id}"
        )
```

## 12. Performance Requirements

### A. Response Time
- **Simple queries**: < 1 second
- **Complex data retrieval**: < 3 seconds
- **Report generation**: < 5 seconds
- **Average response**: < 2 seconds

### B. Availability
- **Uptime**: 99.9% availability
- **Scalability**: Support 1000+ concurrent users
- **Load balancing**: Distributed across multiple instances
- **Failover**: Automatic failover capabilities

### C. Data Accuracy
- **Financial data**: 100% accuracy from source systems
- **Calculations**: Precise mathematical accuracy
- **Currency conversion**: Real-time exchange rates
- **Reporting**: Consistent with system-generated reports

## 13. Integration Points

### A. Core System Integration
- General Ledger for financial data
- Accounts Receivable/Payable modules
- Bank reconciliation tools
- Invoice and billing systems

### B. External Integrations
- Bank feeds for real-time balances
- Tax preparation software
- Audit and compliance tools
- Third-party accounting platforms

### C. User Interface Integration
- Embedded chat widget
- Voice interface support
- Mobile application compatibility
- Accessibility features compliance