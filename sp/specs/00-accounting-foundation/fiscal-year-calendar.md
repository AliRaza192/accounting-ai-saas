# Fiscal Year and Accounting Periods Specification

## Overview

This document defines the framework for managing fiscal years and accounting periods within the Accounting AI SaaS application. It covers the setup, lifecycle management (open, close, reopen), and integration with core accounting processes to ensure data integrity and compliance.

## 1. Fiscal Year Setup

### Fiscal Year Attributes
- **Configurable Start Date**: Allows tenants to define their fiscal year start (e.g., January 1st, April 1st, July 1st, October 1st).
- **Period Support**: Supports both 12 and 13 accounting periods per fiscal year.
- **Multiple Years**: The system must be able to manage multiple fiscal years concurrently, including future, open, and closed years.
- **Base Currency**: Inherits base currency from the tenant for all transactions within the fiscal year.

### Fiscal Year Schema
```sql
CREATE TABLE accounting_core.fiscal_years (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES saas.tenants(id),
  year_number INTEGER NOT NULL, -- e.g., 2024
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  status ENUM('future', 'open', 'closed') DEFAULT 'future',
  closed_at TIMESTAMP,
  closed_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  created_by UUID REFERENCES auth.users(id),
  updated_by UUID REFERENCES auth.users(id),

  UNIQUE(tenant_id, year_number),
  CONSTRAINT chk_fiscal_year_dates CHECK (start_date < end_date)
);

-- Index for efficient lookup
CREATE INDEX idx_fiscal_years_tenant_year ON accounting_core.fiscal_years(tenant_id, year_number);
CREATE INDEX idx_fiscal_years_status ON accounting_core.fiscal_years(status);
```

## 2. Accounting Periods

### Accounting Period Attributes
- **Period Number**: Sequential numbering within a fiscal year (1 to 12 or 1 to 13).
- **Period Name**: Human-readable name (e.g., 'January 2024', 'Q1 FY2024').
- **Start/End Dates**: Defines the exact date range for the period.
- **Status**: Indicates the current state of the period ('future', 'open', 'closed').
- **Closed At/By**: Audit trail for when and by whom a period was closed.

### Accounting Period Schema
```sql
CREATE TABLE accounting_core.periods (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  fiscal_year_id UUID NOT NULL REFERENCES accounting_core.fiscal_years(id) ON DELETE CASCADE,
  period_number INTEGER NOT NULL,
  period_name VARCHAR(50) NOT NULL,
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  status ENUM('future', 'open', 'closed') DEFAULT 'future',
  closed_at TIMESTAMP,
  closed_by UUID REFERENCES auth.users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  created_by UUID REFERENCES auth.users(id),
  updated_by UUID REFERENCES auth.users(id),

  UNIQUE(fiscal_year_id, period_number),
  CONSTRAINT chk_period_dates CHECK (start_date < end_date),
  CONSTRAINT chk_period_number CHECK (period_number > 0 AND period_number <= 13)
);

-- Indexes for efficient lookup
CREATE INDEX idx_periods_fiscal_year_period ON accounting_core.periods(fiscal_year_id, period_number);
CREATE INDEX idx_periods_status ON accounting_core.periods(status);
```

## 3. Period Close Process

### Workflow
1.  **Pre-Close Validations**: System runs a series of checks.
2.  **User Review & Adjustment**: Accountants review and make necessary adjustments.
3.  **Approval**: Requires explicit approval from an authorized user.
4.  **Auto-Generation of Closing Entries**: System generates entries to close temporary accounts.
5.  **Period Lock**: Period status is set to 'closed', preventing further postings.
6.  **Audit Log**: All close actions are logged.

### Validation Checklist (Pre-Close Rules)
- [ ] All bank accounts reconciled up to the period end date.
- [ ] No unposted journal entries exist for the period.
- [ ] No journal entries with a status of 'draft' exist for the period.
- [ ] No outstanding balances in suspense or clearing accounts for the period.
- [ ] All required financial reports (e.g., trial balance, balance sheet, income statement) can be generated successfully for the period.
- [ ] No open invoices or bills with due dates within the closing period that have not been accounted for.
- [ ] All foreign currency accounts have been revalued for unrealized gains/losses.

### Period Lock Enforcement
- Any attempt to create, update, or post a journal entry with an `entry_date` falling within a 'closed' period must be rejected.
- API responses for such attempts will return a `403 Forbidden` error with a specific error code like `PERIOD_LOCKED`.

### Auto-Generate Closing Entries
- **Revenue & Expense Accounts**: Close to an income summary account.
- **Income Summary Account**: Close to Retained Earnings (or owner's equity for non-corporations).
- **Example Closing Entry (Conceptual)**:
    *   Debit: Revenue Accounts (sum of all revenue balances)
    *   Credit: Income Summary Account
    *   Debit: Income Summary Account
    *   Credit: Expense Accounts (sum of all expense balances)
    *   Debit/Credit: Retained Earnings (Net Income/Loss)

## 4. Year-End Close Process

### Workflow
1.  **All Periods Closed**: Ensure all accounting periods within the fiscal year are closed.
2.  **Closing Entries**: Auto-generate final closing entries.
    *   All revenue and expense accounts are closed to a temporary Income Summary account.
    *   The Income Summary account balance is then closed to the Retained Earnings (Equity) account.
3.  **Zero Out Temporary Accounts**: Income Statement accounts (revenue, expenses, draws) are zeroed out.
4.  **Carry Forward Balances**: Balance Sheet accounts (assets, liabilities, equity) balances are carried forward as opening balances for the new fiscal year.
5.  **Create Opening Balances**: For the next fiscal year, system generates opening balance entries for all balance sheet accounts.
6.  **Fiscal Year Lock**: Fiscal year status is set to 'closed'.

### Opening Balance Journal Entry (Conceptual)
- A special journal entry is created on the `start_date` of the new fiscal year, debitting/crediting balance sheet accounts with their respective closing balances from the previous year.

## 5. Reopening Periods

### Procedure
1.  **Supervisor Approval**: Reopening a 'closed' period or fiscal year requires explicit supervisor/admin approval.
2.  **Audit Log Entry**: A detailed audit log entry is created, recording:
    *   Who reopened the period/year.
    *   When it was reopened.
    *   Reason for reopening.
    *   Original close details.
3.  **Notification**: All relevant users (e.g., accountants, finance team) are notified of the reopening.
4.  **Recalculation**: After reopening and any subsequent changes, the system must trigger recalculations for affected reports and potentially re-run closing processes.

### API Endpoints for Fiscal Year & Period Management

#### Fiscal Year Endpoints
```yaml
paths:
  /v1/fiscal-years:
    get:
      summary: List fiscal years
      responses:
        '200':
          description: List of fiscal years
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/FiscalYear'
    post:
      summary: Create new fiscal year
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateFiscalYearRequest'
      responses:
        '201':
          description: Fiscal year created

  /v1/fiscal-years/{yearId}/close:
    post:
      summary: Close a fiscal year
      description: Performs year-end close procedures including closing entries and balance carry-forwards.
      parameters:
        - name: yearId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                reason:
                  type: string
                  description: Reason for closing the fiscal year
      responses:
        '200':
          description: Fiscal year closed successfully
        '400':
          description: Validation error (e.g., periods not closed)

  /v1/fiscal-years/{yearId}/reopen:
    post:
      summary: Reopen a closed fiscal year
      description: Reopens a previously closed fiscal year, requiring supervisor approval and audit logging.
      parameters:
        - name: yearId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [reason]
              properties:
                reason:
                  type: string
                  description: Reason for reopening the fiscal year
      responses:
        '200':
          description: Fiscal year reopened successfully
        '403':
          description: Forbidden (insufficient permissions)
```

#### Period Endpoints
```yaml
paths:
  /v1/periods:
    get:
      summary: List accounting periods
      parameters:
        - name: fiscal_year_id
          in: query
          schema:
            type: string
            format: uuid
          description: Filter by fiscal year
        - name: status
          in: query
          schema:
            type: string
            enum: [future, open, closed]
          description: Filter by period status
      responses:
        '200':
          description: List of accounting periods
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Period'

  /v1/periods/{periodId}/close:
    post:
      summary: Close an accounting period
      description: Executes the period close process, including validations and entry locking.
      parameters:
        - name: periodId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                reason:
                  type: string
                  description: Reason for closing the period
      responses:
        '200':
          description: Period closed successfully
        '400':
          description: Validation error (e.g., unposted entries)

  /v1/periods/{periodId}/reopen:
    post:
      summary: Reopen a closed accounting period
      description: Reopens a previously closed accounting period, requiring supervisor approval and audit logging.
      parameters:
        - name: periodId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [reason]
              properties:
                reason:
                  type: string
                  description: Reason for reopening the period
      responses:
        '200':
          description: Period reopened successfully
        '403':
          description: Forbidden (insufficient permissions)

components:
  schemas:
    FiscalYear:
      type: object
      properties:
        id:
          type: string
          format: uuid
        tenantId:
          type: string
          format: uuid
        yearNumber:
          type: integer
        startDate:
          type: string
          format: date
        endDate:
          type: string
          format: date
        status:
          type: string
          enum: [future, open, closed]
        closedAt:
          type: string
          format: date-time
        closedBy:
          type: string
          format: uuid
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time

    CreateFiscalYearRequest:
      type: object
      required: [yearNumber, startDate, endDate]
      properties:
        yearNumber:
          type: integer
        startDate:
          type: string
          format: date
        endDate:
          type: string
          format: date
        initialStatus:
          type: string
          enum: [future, open]
          default: future

    Period:
      type: object
      properties:
        id:
          type: string
          format: uuid
        fiscalYearId:
          type: string
          format: uuid
        periodNumber:
          type: integer
        periodName:
          type: string
        startDate:
          type: string
          format: date
        endDate:
          type: string
          format: date
        status:
          type: string
          enum: [future, open, closed]
        closedAt:
          type: string
          format: date-time
        closedBy:
          type: string
          format: uuid
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time
```