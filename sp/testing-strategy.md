# Testing Strategy Specification

## Overview

This document outlines the comprehensive testing strategy for the Accounting AI SaaS application, following industry best practices for financial software quality assurance. The strategy emphasizes data integrity, security, and reliability as critical requirements for accounting applications.

## Testing Pyramid

```
        E2E Tests (5%)
       ───────────────
      Integration Tests (15%)
     ────────────────────────
    Unit Tests (80%)
```

## 1. Unit Testing Strategy

### Framework & Tools
- **Backend**: Jest (Node.js) / Pytest (Python)
- **Frontend**: Jest with React Testing Library / Vue Test Utils
- **Coverage Target**: 80% minimum
- **Test File Naming**: `*.test.ts` / `*_test.py`
- **Naming Convention**: Match source file structure

### File Structure Example
```
src/
├── accounting/
│   ├── journal-entry.service.ts
│   └── __tests__/
│       └── journal-entry.service.test.ts
├── auth/
│   ├── user.service.ts
│   └── __tests__/
│       └── user.service.test.ts
└── reports/
    ├── balance-sheet.service.ts
    └── __tests__/
        └── balance-sheet.service.test.ts
```

### Accounting-Specific Unit Tests

#### A. Double-Entry Validation Tests
```typescript
describe('JournalEntryService', () => {
  let journalEntryService: JournalEntryService;

  beforeEach(() => {
    journalEntryService = new JournalEntryService();
  });

  it('should reject unbalanced entry', () => {
    const entry = {
      lines: [
        { account: 'Cash', debit: 1000 },
        { account: 'Revenue', credit: 950 } // Imbalanced!
      ]
    };

    expect(() => journalEntryService.create(entry))
      .toThrow('Journal entry out of balance');
  });

  it('should accept balanced entry', () => {
    const entry = {
      lines: [
        { account: 'Cash', debit: 1000 },
        { account: 'Revenue', credit: 1000 }
      ]
    };

    expect(() => journalEntryService.create(entry))
      .not.toThrow();
  });

  it('should calculate totals correctly', () => {
    const entry = {
      lines: [
        { account: 'Cash', debit: 1000 },
        { account: 'Revenue', credit: 1000 }
      ]
    };

    const result = journalEntryService.create(entry);
    expect(result.totalDebit).toBe(1000);
    expect(result.totalCredit).toBe(1000);
  });

  it('should validate minimum line count', () => {
    const entry = {
      lines: [
        { account: 'Cash', debit: 1000 }
      ] // Only one line, should fail
    };

    expect(() => journalEntryService.create(entry))
      .toThrow('Journal entry must have at least 2 lines');
  });
});
```

#### B. Period Lock Tests
```typescript
describe('PeriodService', () => {
  let periodService: PeriodService;
  let journalEntryService: JournalEntryService;

  beforeEach(() => {
    periodService = new PeriodService();
    journalEntryService = new JournalEntryService();
  });

  it('should prevent posting to closed period', async () => {
    await periodService.closePeriod('2024-01');
    const entry = {
      date: '2024-01-15',
      description: 'Test entry',
      lines: [
        { account: 'Cash', debit: 1000 },
        { account: 'Revenue', credit: 1000 }
      ]
    };

    await expect(journalEntryService.post(entry))
      .rejects.toThrow('Cannot post to closed period');
  });

  it('should allow posting to open period', async () => {
    const entry = {
      date: '2024-02-15',
      description: 'Test entry',
      lines: [
        { account: 'Cash', debit: 1000 },
        { account: 'Revenue', credit: 1000 }
      ]
    };

    await expect(journalEntryService.post(entry))
      .resolves.not.toThrow();
  });
});
```

#### C. Account Balance Validation
```typescript
describe('AccountService', () => {
  let accountService: AccountService;
  let journalEntryService: JournalEntryService;

  it('should update account balances after journal entry', async () => {
    const initialBalance = await accountService.getBalance('cash-account-id');

    const entry = {
      lines: [
        { account: 'cash-account-id', debit: 1000 },
        { account: 'revenue-account-id', credit: 1000 }
      ]
    };

    await journalEntryService.createAndPost(entry);

    const newBalance = await accountService.getBalance('cash-account-id');
    expect(newBalance).toBe(initialBalance + 1000);
  });
});
```

### Unit Test Best Practices
- **Fast execution**: Each test should run under 50ms
- **Isolation**: No external dependencies (use mocks/stubs)
- **Deterministic**: Same input always produces same output
- **Focused**: One assertion per test (AAA pattern)
- **Readable**: Descriptive test names following Given-When-Then

## 2. Integration Testing Strategy

### Framework & Tools
- **API Testing**: Supertest, Axios
- **Database Testing**: Direct DB connections with transaction rollback
- **Container Testing**: Docker Compose for isolated environments

### Key Integration Scenarios

#### A. Complete Invoice-to-Journal Flow
```typescript
describe('Invoice to Journal Integration', () => {
  let dbConnection: Connection;

  beforeAll(async () => {
    dbConnection = await createTestDatabase();
  });

  afterAll(async () => {
    await dbConnection.close();
  });

  it('should create journal entry when invoice is created', async () => {
    const invoiceData = {
      contactId: 'customer-123',
      issueDate: '2024-01-15',
      items: [
        {
          itemName: 'Product A',
          quantity: 1,
          unitPrice: 1000,
          accountId: 'revenue-account-id'
        }
      ]
    };

    const invoice = await invoiceService.create(invoiceData);

    // Verify invoice exists
    expect(invoice).toBeDefined();
    expect(invoice.status).toBe('draft');

    // Submit invoice (creates journal entry)
    const submittedInvoice = await invoiceService.submit(invoice.id);

    // Verify journal entry was created
    const journalEntries = await journalEntryService.getByReference(
      'invoice',
      submittedInvoice.id
    );

    expect(journalEntries).toHaveLength(2); // Revenue + Receivable
    expect(journalEntries[0].totalDebit).toEqual(journalEntries[0].totalCredit);
  });

  it('should create payment journal entry when payment is received', async () => {
    const invoice = await invoiceService.create(validInvoiceData);
    const submittedInvoice = await invoiceService.submit(invoice.id);

    const payment = await paymentService.create({
      invoiceId: submittedInvoice.id,
      amount: submittedInvoice.totalAmount,
      paymentMethod: 'check'
    });

    // Verify payment journal entry
    const paymentEntries = await journalEntryService.getByReference(
      'payment',
      payment.id
    );

    expect(paymentEntries).toHaveLength(2); // Cash + Receivable
  });
});
```

#### B. Multi-Tenant Isolation Verification
```typescript
describe('Multi-Tenant Isolation', () => {
  let tenant1Connection: Connection;
  let tenant2Connection: Connection;

  it('should not allow tenant1 to access tenant2 data', async () => {
    // Create data for tenant1
    await createTestDataForTenant('tenant1', tenant1Connection);

    // Create data for tenant2
    await createTestDataForTenant('tenant2', tenant2Connection);

    // Attempt to access tenant2's data from tenant1 context
    const tenant1Service = new JournalEntryService('tenant1');
    const tenant2Entries = await tenant1Service.getAll({
      tenantId: 'tenant2'
    });

    // Should return empty or throw error depending on implementation
    expect(tenant2Entries).toHaveLength(0);
  });
});
```

#### C. Bank Reconciliation Workflow
```typescript
describe('Bank Reconciliation', () => {
  it('should match transactions and mark as reconciled', async () => {
    // Create bank transactions
    const bankTransactions = await bankService.importTransactions([
      {
        description: 'Customer Payment',
        amount: 1000,
        date: '2024-01-15'
      }
    ]);

    // Create corresponding journal entries
    await journalEntryService.create({
      description: 'Customer Payment',
      date: '2024-01-15',
      lines: [
        { account: 'cash-account', debit: 1000 },
        { account: 'receivable-account', credit: 1000 }
      ]
    });

    // Perform reconciliation
    const reconciliationResult = await reconciliationService.reconcile({
      bankAccountId: 'bank-acc-123',
      bankTransactions: bankTransactions.map(bt => bt.id),
      journalEntries: ['journal-entry-id']
    });

    expect(reconciliationResult.success).toBe(true);
    expect(reconciliationResult.matchedPairs).toHaveLength(1);
  });
});
```

### Integration Test Best Practices
- **Database transactions**: Wrap tests in transactions that are rolled back
- **Real dependencies**: Test with actual database, not mocks
- **Environment setup**: Use Docker Compose for consistent test environments
- **Clean state**: Reset database state between test runs

## 3. End-to-End (E2E) Testing Strategy

### Framework & Tools
- **Primary**: Playwright
- **Alternative**: Cypress
- **Mobile**: Appium (for mobile apps)

### Critical User Journeys

#### A. New User Onboarding Flow
```typescript
// playwright/e2e/onboarding.spec.ts
import { test, expect } from '@playwright/test';

test.describe('New User Onboarding', () => {
  test('complete signup to first invoice flow', async ({ page }) => {
    // Navigate to signup page
    await page.goto('/signup');

    // Fill signup form
    await page.fill('input[name="email"]', 'test@example.com');
    await page.fill('input[name="password"]', 'SecurePassword123!');
    await page.click('button[type="submit"]');

    // Verify email verification
    await page.waitForURL('/verify-email');
    await page.click('button[data-testid="resend-verification"]');

    // Complete company setup
    await page.fill('input[name="companyName"]', 'Test Company');
    await page.fill('input[name="businessType"]', 'service');
    await page.click('button[type="submit"]');

    // Navigate to contacts and create customer
    await page.click('nav a[href="/contacts"]');
    await page.click('button[data-testid="new-contact"]');
    await page.fill('input[name="companyName"]', 'ACME Corp');
    await page.fill('input[name="email"]', 'billing@acme.com');
    await page.click('button[type="submit"]');

    // Create first invoice
    await page.click('nav a[href="/invoices"]');
    await page.click('button[data-testid="new-invoice"]');
    await page.click('select[name="contactId"]');
    await page.click('text="ACME Corp"');

    // Add line items
    await page.click('button[data-testid="add-item"]');
    await page.fill('input[name="itemName"]', 'Consulting Services');
    await page.fill('input[name="quantity"]', '1');
    await page.fill('input[name="unitPrice"]', '1000');

    // Save and submit invoice
    await page.click('button[data-testid="save-draft"]');
    await page.click('button[data-testid="submit-invoice"]');

    // Verify invoice was created successfully
    await expect(page.locator('text="Invoice submitted successfully"')).toBeVisible();
  });
});
```

#### B. Month-End Close Workflow
```typescript
test.describe('Month-End Close', () => {
  test('perform month-end close process', async ({ page, context }) => {
    // Login as admin user
    await loginAsAdmin(page);

    // Navigate to period close
    await page.click('nav a[href="/reports"]');
    await page.click('text="Period Close"');

    // Select period to close
    await page.click('select[name="period"]');
    await page.click('option[value="2024-01"]');

    // Run pre-close validations
    await page.click('button[data-testid="run-validations"]');
    await expect(page.locator('text="All validations passed"')).toBeVisible();

    // Generate adjusting entries
    await page.click('button[data-testid="generate-adjustments"]');
    await expect(page.locator('text="Adjusting entries generated"')).toBeVisible();

    // Review and post adjustments
    await page.click('a[href="/journal-entries?type=adjusting"]');
    await page.click('button[data-testid="post-all"]');

    // Close the period
    await page.click('button[data-testid="close-period"]');

    // Verify period is closed
    await expect(page.locator('text="Period 2024-01 is now closed"')).toBeVisible();
  });
});
```

#### C. Financial Report Generation
```typescript
test.describe('Financial Reports', () => {
  test('generate and export balance sheet', async ({ page }) => {
    // Navigate to reports
    await page.goto('/reports');

    // Click on balance sheet
    await page.click('text="Balance Sheet"');

    // Configure report parameters
    await page.fill('input[name="asOfDate"]', '2024-01-31');

    // Generate report
    await page.click('button[data-testid="generate-report"]');

    // Wait for report to load
    await expect(page.locator('table.balance-sheet')).toBeVisible();

    // Verify report structure
    await expect(page.locator('th:has-text("Assets")')).toBeVisible();
    await expect(page.locator('th:has-text("Liabilities")')).toBeVisible();
    await expect(page.locator('th:has-text("Equity")')).toBeVisible();

    // Export to PDF
    await page.click('button[data-testid="export-pdf"]');
    await expect(page.locator('text="Report exported successfully"')).toBeVisible();
  });
});
```

### E2E Test Best Practices
- **Real browsers**: Test across Chrome, Firefox, Safari
- **Slow motion**: Use for debugging (not production)
- **Wait strategies**: Explicit waits instead of arbitrary timeouts
- **Page Object Model**: Organize tests with page objects
- **Parallel execution**: Run tests in parallel to save time

## 4. Performance Testing Strategy

### Load Testing Scenarios
- **Concurrent Users**: 1000 simultaneous users
- **Peak Load**: 2000 requests per minute
- **Spike Testing**: Sudden load increases

### Performance Benchmarks
| Metric | Target | 95th Percentile |
|--------|--------|-----------------|
| API Response Time | <100ms | <200ms |
| Database Query Time | <50ms | <100ms |
| Page Load Time | <2s | <3s |
| Report Generation | <5s | <10s |

### Performance Test Implementation
```typescript
// k6 load test script
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '5m', target: 100 },    // Ramp up to 100 users
    { duration: '10m', target: 100 },   // Maintain 100 users
    { duration: '5m', target: 200 },    // Spike to 200 users
    { duration: '10m', target: 200 },   // Maintain 200 users
    { duration: '5m', target: 0 },      // Ramp down to 0
  ],
};

export default function() {
  const baseUrl = 'https://api.accountingai.pro/v1';

  // Test journal entry creation
  const journalEntry = {
    entry_date: '2024-01-15',
    description: 'Load test entry',
    lines: [
      { account_id: 'cash-account', debit: 1000 },
      { account_id: 'revenue-account', credit: 1000 }
    ]
  };

  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer ' + __ENV.API_TOKEN,
    },
  };

  const response = http.post(
    `${baseUrl}/journal-entries`,
    JSON.stringify(journalEntry),
    params
  );

  check(response, {
    'journal entry created successfully': (r) => r.status === 201,
    'response time under 200ms': (r) => r.timings.duration < 200,
  });

  sleep(1);
}
```

## 5. Security Testing Strategy

### Security Test Categories

#### A. SQL Injection Prevention
```typescript
describe('SQL Injection Prevention', () => {
  it('should prevent SQL injection in search queries', async () => {
    const maliciousQuery = "'; DROP TABLE users; --";

    await expect(() =>
      userService.search(maliciousQuery)
    ).rejects.toThrow();
  });
});

// Database layer validation
const safeQuery = (query: string, params: any[]) => {
  // Use parameterized queries
  return db.query(query, params);
};
```

#### B. XSS Prevention
```typescript
describe('XSS Prevention', () => {
  it('should sanitize user input', async () => {
    const maliciousInput = '<script>alert("XSS")</script>';

    const result = await journalEntryService.create({
      description: maliciousInput,
      lines: []
    });

    // Verify output is sanitized
    expect(result.description).not.toContain('<script>');
  });
});
```

#### C. CSRF Protection
```typescript
describe('CSRF Protection', () => {
  it('should validate CSRF tokens', async () => {
    const response = await request(app)
      .post('/api/journal-entries')
      .send(validJournalEntryData)
      .expect(403); // Missing CSRF token

    expect(response.body.error.code).toBe('CSRF_TOKEN_MISSING');
  });
});
```

#### D. Multi-Tenant Data Isolation
```typescript
describe('Multi-Tenant Isolation', () => {
  it('should enforce tenant boundaries', async () => {
    // User from tenant A tries to access tenant B's data
    const userAToken = await authenticateUser('user-a@tenant-a.com');
    const response = await request(app)
      .get('/api/journal-entries?tenantId=tenant-b-id')
      .set('Authorization', `Bearer ${userAToken}`)
      .expect(403);

    expect(response.body.error.code).toBe('TENANT_ACCESS_DENIED');
  });
});
```

### Security Checklist
- [ ] Input validation and sanitization
- [ ] Authentication and authorization
- [ ] Session management
- [ ] Data encryption (at rest and in transit)
- [ ] API rate limiting
- [ ] Secure configuration
- [ ] Dependency vulnerability scanning
- [ ] Regular security audits

## 6. CI/CD Integration

### GitHub Actions Workflow
```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linting
        run: npm run lint

      - name: Run unit tests
        run: npm run test:unit
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db

      - name: Generate coverage report
        run: npm run test:coverage

      - name: Check coverage threshold
        run: |
          # Verify coverage is above 80%
          npx jest --coverage --coverage-threshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
```

### Quality Gates
- **Code Coverage**: Minimum 80% for all test types
- **Performance**: All API endpoints must respond in <100ms (95th percentile)
- **Security**: No high-severity vulnerabilities allowed
- **Build Time**: Total test suite should complete within 10 minutes

## 7. Test Data Management

### Data Factories
```typescript
// factories/user.factory.ts
import { faker } from '@faker-js/faker';

export const createUser = (overrides = {}) => ({
  email: faker.internet.email(),
  fullName: faker.name.fullName(),
  password: faker.internet.password(),
  isActive: true,
  ...overrides
});

// factories/journal-entry.factory.ts
export const createJournalEntry = (overrides = {}) => ({
  entryDate: faker.date.past(),
  description: faker.lorem.sentence(),
  referenceType: faker.helpers.arrayElement(['invoice', 'bill', 'payment', 'manual']),
  lines: [
    {
      accountId: faker.datatype.uuid(),
      debit: faker.finance.amount(100, 10000),
      credit: 0
    },
    {
      accountId: faker.datatype.uuid(),
      debit: 0,
      credit: faker.finance.amount(100, 10000)
    }
  ],
  ...overrides
});
```

### Test Fixtures
```typescript
// fixtures/chart-of-accounts.fixture.ts
export const standardChartOfAccounts = [
  {
    id: 'asset-cash',
    code: '1000',
    name: 'Cash',
    accountType: 'asset',
    normalBalance: 'debit'
  },
  {
    id: 'asset-accounts-receivable',
    code: '1100',
    name: 'Accounts Receivable',
    accountType: 'asset',
    normalBalance: 'debit'
  },
  {
    id: 'revenue-service-income',
    code: '4000',
    name: 'Service Income',
    accountType: 'revenue',
    normalBalance: 'credit'
  }
];
```

### Database Seeding
```typescript
// scripts/seeds/development.seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function seedDevelopment() {
  // Create test tenant
  const tenant = await prisma.tenant.create({
    data: {
      name: 'Development Tenant',
      subdomain: 'dev',
      tier: 'professional'
    }
  });

  // Create test user
  const user = await prisma.user.create({
    data: {
      email: 'admin@dev.example.com',
      fullName: 'Dev Admin',
      tenantId: tenant.id,
      passwordHash: '$2b$10$...' // Hashed password
    }
  });

  // Create standard chart of accounts
  await prisma.chartOfAccount.createMany({
    data: standardChartOfAccounts.map(account => ({
      ...account,
      tenantId: tenant.id
    }))
  });

  console.log('Development data seeded successfully');
}

seedDevelopment()
  .catch(e => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

## 8. Monitoring and Reporting

### Test Metrics Dashboard
- **Pass Rate**: Overall test success percentage
- **Execution Time**: Average and trending test times
- **Coverage Trends**: Historical coverage data
- **Flaky Tests**: Tests that occasionally fail
- **Performance Regressions**: Performance degradation over time

### Alerting Strategy
- **Critical Failures**: Immediate alerts for broken builds
- **Performance Degradation**: Alerts when response times exceed thresholds
- **Coverage Drops**: Notifications when test coverage decreases
- **Security Issues**: Immediate alerts for security test failures

This comprehensive testing strategy ensures the Accounting AI SaaS application maintains high quality, security, and reliability standards while supporting rapid development cycles.