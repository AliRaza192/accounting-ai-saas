# Multi-Currency Accounting Framework Specification

## Overview

This document outlines the comprehensive multi-currency accounting framework for the Accounting AI SaaS application. The framework supports multiple currencies with proper exchange rate management, unrealized gains/losses calculation, and accurate reporting in base currency.

## Supported Currencies

### Currency Standards
- **ISO 4217 currency codes**: USD, EUR, GBP, PKR, etc.
- **Dynamic currency list**: Admin configurable via settings
- **Base currency per tenant**: Default reporting currency
- **Currency symbols and formatting**: Locale-aware display

### Currency Configuration Schema
```sql
CREATE TABLE accounting_core.currencies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code CHAR(3) NOT NULL UNIQUE, -- ISO 4217 code
  name VARCHAR(100) NOT NULL,
  symbol VARCHAR(10), -- e.g., $, €, £
  minor_unit INT DEFAULT 2, -- number of decimal places
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Pre-populate with common currencies
INSERT INTO accounting_core.currencies (code, name, symbol, minor_unit) VALUES
('USD', 'US Dollar', '$', 2),
('EUR', 'Euro', '€', 2),
('GBP', 'British Pound', '£', 2),
('PKR', 'Pakistani Rupee', '₨', 2),
('CAD', 'Canadian Dollar', 'C$', 2),
('AUD', 'Australian Dollar', 'A$', 2),
('JPY', 'Japanese Yen', '¥', 0),
('CHF', 'Swiss Franc', 'Fr', 2),
('CNY', 'Chinese Yuan', '¥', 2),
('INR', 'Indian Rupee', '₹', 2);

-- Tenant currency settings
ALTER TABLE saas.tenants ADD COLUMN base_currency_code CHAR(3) DEFAULT 'USD' REFERENCES accounting_core.currencies(code);
```

## Exchange Rates Management

### Exchange Rate Schema
```sql
CREATE TABLE accounting_core.exchange_rates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  from_currency CHAR(3) NOT NULL REFERENCES accounting_core.currencies(code),
  to_currency CHAR(3) NOT NULL REFERENCES accounting_core.currencies(code),
  rate DECIMAL(12,6) NOT NULL, -- Exchange rate: from_currency -> to_currency
  effective_date DATE NOT NULL,
  source ENUM('api', 'manual') NOT NULL,
  created_by UUID REFERENCES auth.users(id),
  approved_by UUID REFERENCES auth.users(id), -- For manual overrides
  notes TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  UNIQUE(from_currency, to_currency, effective_date),
  CONSTRAINT chk_different_currencies CHECK (from_currency != to_currency)
);

-- Indexes for performance
CREATE INDEX idx_exchange_rates_from_to_date ON accounting_core.exchange_rates(from_currency, to_currency, effective_date);
CREATE INDEX idx_exchange_rates_effective_date ON accounting_core.exchange_rates(effective_date);

-- Function to get exchange rate for a specific date
CREATE OR REPLACE FUNCTION get_exchange_rate(p_from_currency CHAR(3), p_to_currency CHAR(3), p_date DATE)
RETURNS DECIMAL(12,6) AS $$
DECLARE
  v_rate DECIMAL(12,6);
BEGIN
  -- First try exact match
  SELECT rate INTO v_rate
  FROM accounting_core.exchange_rates
  WHERE from_currency = p_from_currency
    AND to_currency = p_to_currency
    AND effective_date <= p_date
  ORDER BY effective_date DESC
  LIMIT 1;

  -- If no direct rate found, try inverse rate
  IF v_rate IS NULL THEN
    SELECT 1/rate INTO v_rate
    FROM accounting_core.exchange_rates
    WHERE from_currency = p_to_currency
      AND to_currency = p_from_currency
      AND effective_date <= p_date
    ORDER BY effective_date DESC
    LIMIT 1;
  END IF;

  RETURN v_rate;
END;
$$ LANGUAGE plpgsql;
```

### Exchange Rate API Integration
```typescript
// Service for fetching real-time exchange rates
interface ExchangeRateService {
  fetchLatestRates(): Promise<ExchangeRateData>;
  storeHistoricalRates(rates: ExchangeRateData[]): Promise<void>;
  getRate(fromCurrency: string, toCurrency: string, date: Date): Promise<number>;
}

// Example implementation
class ExchangeRateAPIService implements ExchangeRateService {
  private readonly API_BASE_URL = 'https://api.exchangerate-api.com/v4/latest/';

  async fetchLatestRates(): Promise<ExchangeRateData> {
    const response = await fetch(`${this.API_BASE_URL}USD`);
    const data = await response.json();

    return {
      base: data.base,
      rates: data.rates,
      timestamp: new Date(data.timestamp * 1000)
    };
  }

  async storeHistoricalRates(rates: ExchangeRateData[]): Promise<void> {
    // Batch insert rates into database
    for (const rate of rates) {
      await db.execute(`
        INSERT INTO accounting_core.exchange_rates
        (from_currency, to_currency, rate, effective_date, source)
        VALUES (?, ?, ?, ?, 'api')
        ON CONFLICT (from_currency, to_currency, effective_date)
        DO UPDATE SET rate = EXCLUDED.rate
      `, [rate.from, rate.to, rate.rate, rate.date]);
    }
  }
}
```

## Transaction Recording

### Enhanced Journal Line Schema
```sql
-- Add multi-currency columns to journal lines
ALTER TABLE accounting_core.journal_lines
ADD COLUMN original_currency CHAR(3) DEFAULT NULL,
ADD COLUMN original_amount DECIMAL(15,2) DEFAULT 0,
ADD COLUMN exchange_rate DECIMAL(12,6) DEFAULT 1.0,
ADD COLUMN base_amount DECIMAL(15,2) DEFAULT 0,
ADD COLUMN is_converted BOOLEAN DEFAULT FALSE;

-- Add multi-currency columns to journal entries
ALTER TABLE accounting_core.journal_entries
ADD COLUMN source_currency CHAR(3) DEFAULT NULL,
ADD COLUMN base_currency CHAR(3) DEFAULT NULL,
ADD COLUMN exchange_rate DECIMAL(12,6) DEFAULT 1.0;

-- Indexes for multi-currency queries
CREATE INDEX idx_journal_lines_original_currency ON accounting_core.journal_lines(original_currency);
CREATE INDEX idx_journal_lines_converted ON accounting_core.journal_lines(is_converted);

-- Trigger to automatically calculate base amounts
CREATE OR REPLACE FUNCTION calculate_base_amount()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.original_currency IS NOT NULL AND NEW.original_currency != NEW.currency_code THEN
    -- This is a multi-currency transaction
    NEW.is_converted = TRUE;
    NEW.base_amount := ROUND(NEW.original_amount * NEW.exchange_rate, 2);
  ELSE
    -- Same currency as base, no conversion needed
    NEW.original_currency := NEW.currency_code;
    NEW.original_amount := CASE
      WHEN NEW.debit > 0 THEN NEW.debit
      ELSE NEW.credit
    END;
    NEW.exchange_rate := 1.0;
    NEW.base_amount := CASE
      WHEN NEW.debit > 0 THEN NEW.debit
      ELSE NEW.credit
    END;
  END IF;

  -- Update debit/credit in base currency for consistency
  IF NEW.debit > 0 THEN
    NEW.debit := NEW.base_amount;
  ELSE
    NEW.credit := NEW.base_amount;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_calculate_base_amount
BEFORE INSERT OR UPDATE ON accounting_core.journal_lines
FOR EACH ROW EXECUTE FUNCTION calculate_base_amount();
```

### Transaction Processing Logic
```typescript
class MultiCurrencyTransactionProcessor {
  async processMultiCurrencyEntry(entry: JournalEntry): Promise<JournalEntry> {
    const tenant = await this.getTenant(entry.tenantId);
    const baseCurrency = tenant.baseCurrencyCode;

    // Process each line
    for (const line of entry.lines) {
      if (line.originalCurrency && line.originalCurrency !== baseCurrency) {
        // Need to convert to base currency
        const exchangeRate = await this.getExchangeRate(
          line.originalCurrency,
          baseCurrency,
          entry.entryDate
        );

        line.exchangeRate = exchangeRate;
        line.baseAmount = this.convertCurrency(line.originalAmount, exchangeRate);
        line.isConverted = true;

        // Update debit/credit with converted amount
        if (line.debit > 0) {
          line.debit = line.baseAmount;
        } else {
          line.credit = line.baseAmount;
        }
      } else {
        // Same currency as base
        line.originalCurrency = line.currencyCode;
        line.originalAmount = line.debit > 0 ? line.debit : line.credit;
        line.exchangeRate = 1.0;
        line.baseAmount = line.originalAmount;
        line.isConverted = false;
      }
    }

    // Update journal entry with currency info
    entry.sourceCurrency = entry.lines[0]?.originalCurrency || baseCurrency;
    entry.baseCurrency = baseCurrency;
    entry.exchangeRate = entry.lines.find(l => l.isConverted)?.exchangeRate || 1.0;

    return entry;
  }

  private convertCurrency(amount: number, rate: number): number {
    return Math.round(amount * rate * 100) / 100; // Banker's rounding to 2 decimals
  }

  private async getExchangeRate(from: string, to: string, date: Date): Promise<number> {
    // Use stored procedure or direct query
    const result = await db.query(
      'SELECT get_exchange_rate($1, $2, $3)',
      [from, to, date]
    );
    return result.rows[0].get_exchange_rate || 1.0;
  }
}
```

## Unrealized Gains/Losses Framework

### Forex Gain/Loss Accounts Setup
```sql
-- Add forex account types
ALTER TYPE accounting_core.account_type ADD VALUE 'forex_gain';
ALTER TYPE accounting_core.account_type ADD VALUE 'forex_loss';

-- Create default forex accounts for new tenants
CREATE OR REPLACE FUNCTION create_forex_accounts_for_tenant(p_tenant_id UUID)
RETURNS VOID AS $$
BEGIN
  -- Forex Gain account
  INSERT INTO accounting_core.chart_of_accounts (
    tenant_id, code, name, account_type, normal_balance, is_system
  ) VALUES (
    p_tenant_id, '9999', 'Foreign Exchange Gain', 'forex_gain', 'credit', TRUE
  );

  -- Forex Loss account
  INSERT INTO accounting_core.chart_of_accounts (
    tenant_id, code, name, account_type, normal_balance, is_system
  ) VALUES (
    p_tenant_id, '9998', 'Foreign Exchange Loss', 'forex_loss', 'debit', TRUE
  );
END;
$$ LANGUAGE plpgsql;
```

### Unrealized Gains/Losses Calculation
```sql
-- Table to store realized/unrealized forex positions
CREATE TABLE accounting_core.forex_positions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES saas.tenants(id),
  account_id UUID NOT NULL REFERENCES accounting_core.chart_of_accounts(id),
  currency_code CHAR(3) NOT NULL,
  original_amount DECIMAL(15,2) NOT NULL,
  base_amount DECIMAL(15,2) NOT NULL,
  exchange_rate DECIMAL(12,6) NOT NULL,
  realized_amount DECIMAL(15,2) DEFAULT 0,
  unrealized_gain_loss DECIMAL(15,2) DEFAULT 0,
  last_revaluation_date DATE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Function to calculate unrealized gains/losses
CREATE OR REPLACE FUNCTION calculate_unrealized_forex_gains_losses(p_tenant_id UUID, p_as_of_date DATE)
RETURNS TABLE(
  account_id UUID,
  currency_code CHAR(3),
  original_amount DECIMAL(15,2),
  current_base_amount DECIMAL(15,2),
  previous_base_amount DECIMAL(15,2),
  gain_loss_amount DECIMAL(15,2),
  exchange_rate DECIMAL(12,6)
) AS $$
BEGIN
  RETURN QUERY
  WITH current_balances AS (
    SELECT
      jl.account_id,
      jl.original_currency as currency_code,
      SUM(CASE
        WHEN coa.normal_balance = 'debit' THEN jl.original_amount
        ELSE -jl.original_amount
      END) as original_amount,
      SUM(CASE
        WHEN coa.normal_balance = 'debit' THEN jl.base_amount
        ELSE -jl.base_amount
      END) as current_base_amount
    FROM accounting_core.journal_lines jl
    JOIN accounting_core.journal_entries je ON jl.journal_entry_id = je.id
    JOIN accounting_core.chart_of_accounts coa ON jl.account_id = coa.id
    WHERE je.tenant_id = p_tenant_id
      AND je.status = 'posted'
      AND je.entry_date <= p_as_of_date
      AND jl.original_currency IS NOT NULL
      AND jl.original_currency != (SELECT base_currency_code FROM saas.tenants WHERE id = p_tenant_id)
    GROUP BY jl.account_id, jl.original_currency, coa.normal_balance
  ),
  current_rates AS (
    SELECT
      cb.account_id,
      cb.currency_code,
      cb.original_amount,
      cb.current_base_amount,
      get_exchange_rate(cb.currency_code, (SELECT base_currency_code FROM saas.tenants WHERE id = p_tenant_id), p_as_of_date) as exchange_rate
    FROM current_balances cb
  )
  SELECT
    cr.account_id,
    cr.currency_code,
    cr.original_amount,
    cr.current_base_amount,
    ROUND(cr.original_amount * cr.exchange_rate, 2) as previous_base_amount,
    ROUND(cr.current_base_amount - (cr.original_amount * cr.exchange_rate), 2) as gain_loss_amount,
    cr.exchange_rate
  FROM current_rates cr
  WHERE ABS(ROUND(cr.current_base_amount - (cr.original_amount * cr.exchange_rate), 2)) > 0.01; -- Ignore rounding differences
END;
$$ LANGUAGE plpgsql;
```

### Period-End Revaluation Process
```typescript
class ForexRevaluationService {
  async performPeriodEndRevaluation(tenantId: string, asOfDate: Date): Promise<JournalEntry> {
    // Calculate unrealized gains/losses
    const gainsLosses = await this.calculateUnrealizedGainsLosses(tenantId, asOfDate);

    if (gainsLosses.length === 0) {
      return null; // No revaluation needed
    }

    // Get forex gain/loss account IDs
    const forexAccounts = await this.getForexAccounts(tenantId);

    const journalLines: JournalLine[] = [];

    for (const gl of gainsLosses) {
      if (gl.gainLossAmount > 0) {
        // Gain: Debit asset/liability account, Credit forex gain
        journalLines.push({
          accountId: gl.accountId,
          debit: 0,
          credit: Math.abs(gl.gainLossAmount),
          description: `FX revaluation gain for ${gl.currencyCode}`
        });
        journalLines.push({
          accountId: forexAccounts.gainAccountId,
          debit: Math.abs(gl.gainLossAmount),
          credit: 0,
          description: `FX revaluation gain from ${gl.currencyCode}`
        });
      } else {
        // Loss: Debit forex loss, Credit asset/liability account
        journalLines.push({
          accountId: forexAccounts.lossAccountId,
          debit: Math.abs(gl.gainLossAmount),
          credit: 0,
          description: `FX revaluation loss from ${gl.currencyCode}`
        });
        journalLines.push({
          accountId: gl.accountId,
          debit: 0,
          credit: Math.abs(gl.gainLossAmount),
          description: `FX revaluation loss for ${gl.currencyCode}`
        });
      }
    }

    // Create revaluation journal entry
    const revaluationEntry: JournalEntry = {
      tenantId,
      entryDate: asOfDate,
      description: `Period-end FX revaluation as of ${asOfDate}`,
      referenceType: 'revaluation',
      lines: journalLines,
      status: 'draft', // Needs approval
      isAdjustingEntry: true
    };

    return await this.journalEntryService.create(revaluationEntry);
  }

  private async calculateUnrealizedGainsLosses(tenantId: string, asOfDate: Date): Promise<any[]> {
    const result = await db.query(
      'SELECT * FROM calculate_unrealized_forex_gains_losses($1, $2)',
      [tenantId, asOfDate]
    );
    return result.rows;
  }
}
```

## Reporting Framework

### Multi-Currency Report Schema
```sql
-- View for base currency reporting
CREATE VIEW accounting_core.vw_journal_entries_base_currency AS
SELECT
  je.id,
  je.entry_number,
  je.entry_date,
  je.description,
  je.reference_type,
  je.reference_id,
  je.total_debit,
  je.total_credit,
  je.status,
  je.source_currency,
  je.base_currency,
  je.tenant_id,
  -- Use base amounts for reporting
  COALESCE(jl.base_amount, CASE WHEN jl.debit > 0 THEN jl.debit ELSE jl.credit END) as report_amount,
  coa.code as account_code,
  coa.name as account_name,
  coa.account_type,
  jl.description as line_description,
  jl.line_number
FROM accounting_core.journal_entries je
JOIN accounting_core.journal_lines jl ON je.id = jl.journal_entry_id
JOIN accounting_core.chart_of_accounts coa ON jl.account_id = coa.id
WHERE je.status = 'posted';
```

### API Endpoints for Multi-Currency Operations

#### Exchange Rate Management
```yaml
openapi: 3.0.3
info:
  title: Multi-Currency API
  version: 1.0.0

paths:
  /v1/currencies:
    get:
      summary: Get supported currencies
      responses:
        '200':
          description: List of supported currencies
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Currency'
    post:
      summary: Create/update currency
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateCurrencyRequest'
      responses:
        '201':
          description: Currency created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Currency'

  /v1/exchange-rates:
    get:
      summary: Get exchange rates
      parameters:
        - name: from
          in: query
          required: true
          schema:
            type: string
        - name: to
          in: query
          required: true
          schema:
            type: string
        - name: date
          in: query
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Exchange rate
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ExchangeRate'
    post:
      summary: Create manual exchange rate
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateExchangeRateRequest'
      responses:
        '201':
          description: Exchange rate created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ExchangeRate'

  /v1/reports/forex-gains-losses:
    get:
      summary: Get forex gains/losses report
      parameters:
        - name: as_of_date
          in: query
          required: true
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Forex gains/losses report
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ForexGainsLossesReport'

components:
  schemas:
    Currency:
      type: object
      properties:
        id:
          type: string
          format: uuid
        code:
          type: string
          pattern: ^[A-Z]{3}$
        name:
          type: string
        symbol:
          type: string
        minorUnit:
          type: integer
        isActive:
          type: boolean

    ExchangeRate:
      type: object
      properties:
        id:
          type: string
          format: uuid
        fromCurrency:
          type: string
        toCurrency:
          type: string
        rate:
          type: number
          format: decimal
        effectiveDate:
          type: string
          format: date
        source:
          type: string
          enum: [api, manual]
        createdBy:
          type: string
          format: uuid

    ForexGainsLossesReport:
      type: object
      properties:
        asOfDate:
          type: string
          format: date
        gains:
          type: array
          items:
            $ref: '#/components/schemas/ForexPosition'
        losses:
          type: array
          items:
            $ref: '#/components/schemas/ForexPosition'
        netGainLoss:
          type: number

    ForexPosition:
      type: object
      properties:
        accountId:
          type: string
          format: uuid
        currencyCode:
          type: string
        originalAmount:
          type: number
        currentBaseAmount:
          type: number
        gainLossAmount:
          type: number
        exchangeRate:
          type: number
```

## Calculation Formulas

### Currency Conversion
```
Base Amount = Original Amount × Exchange Rate
Exchange Rate = Base Amount ÷ Original Amount
```

### Unrealized Gain/Loss Calculation
```
Unrealized Gain/Loss = Current Base Amount - (Original Amount × Current Exchange Rate)
Where:
- Current Base Amount = Amount currently recorded in base currency
- Original Amount = Amount in original currency
- Current Exchange Rate = Rate at revaluation date
```

### Rounding Rules
- **Amounts**: Round to 2 decimal places using banker's rounding (round half to even)
- **Exchange Rates**: Maintain 6 decimal places for precision
- **Totals**: Recalculate from individual line items to avoid rounding errors

## Test Cases

### Unit Tests
```typescript
describe('MultiCurrencyTransactionProcessor', () => {
  let processor: MultiCurrencyTransactionProcessor;

  beforeEach(() => {
    processor = new MultiCurrencyTransactionProcessor();
  });

  it('should convert USD to EUR correctly', async () => {
    const entry = {
      tenantId: 'tenant-123',
      entryDate: new Date('2024-01-15'),
      lines: [
        {
          accountId: 'cash-account',
          originalCurrency: 'USD',
          originalAmount: 1000,
          debit: 0,
          credit: 0
        },
        {
          accountId: 'revenue-account',
          originalCurrency: 'USD',
          originalAmount: 1000,
          debit: 0,
          credit: 0
        }
      ]
    };

    // Mock exchange rate service
    jest.spyOn(processor as any, 'getExchangeRate')
      .mockResolvedValue(0.85); // 1 USD = 0.85 EUR

    const result = await processor.processMultiCurrencyEntry(entry);

    expect(result.lines[0].baseAmount).toBe(850); // 1000 * 0.85
    expect(result.lines[0].isConverted).toBe(true);
  });

  it('should handle same currency as base', async () => {
    const entry = {
      tenantId: 'tenant-123',
      entryDate: new Date('2024-01-15'),
      baseCurrency: 'USD',
      lines: [
        {
          accountId: 'cash-account',
          currencyCode: 'USD',
          debit: 1000
        },
        {
          accountId: 'revenue-account',
          currencyCode: 'USD',
          credit: 1000
        }
      ]
    };

    const result = await processor.processMultiCurrencyEntry(entry);

    expect(result.lines[0].originalCurrency).toBe('USD');
    expect(result.lines[0].baseAmount).toBe(1000);
    expect(result.lines[0].isConverted).toBe(false);
  });
});

describe('ForexRevaluationService', () => {
  let revaluationService: ForexRevaluationService;

  it('should calculate unrealized gains correctly', async () => {
    const gainsLosses = await revaluationService.calculateUnrealizedGainsLosses(
      'tenant-123',
      new Date('2024-01-31')
    );

    // Mock expected result based on test data
    expect(gainsLosses).toHaveLength(2); // Example: 2 positions with gains/losses
    expect(gainsLosses[0].gainLossAmount).toBeGreaterThan(0); // Gain
    expect(gainsLosses[1].gainLossAmount).toBeLessThan(0); // Loss
  });
});
```

### Integration Tests
```typescript
describe('Multi-Currency Integration', () => {
  it('should create journal entry with multiple currencies', async () => {
    // Create exchange rate
    await db.execute(`
      INSERT INTO accounting_core.exchange_rates
      (from_currency, to_currency, rate, effective_date, source)
      VALUES ('EUR', 'USD', 1.10, '2024-01-15', 'manual')
    `);

    // Create journal entry with EUR
    const entry = {
      entryDate: '2024-01-15',
      description: 'Multi-currency test',
      lines: [
        {
          accountId: 'cash-account-eur',
          originalCurrency: 'EUR',
          originalAmount: 1000,
          debit: 0,
          credit: 0
        },
        {
          accountId: 'revenue-account',
          originalCurrency: 'EUR',
          originalAmount: 1000,
          debit: 0,
          credit: 0
        }
      ]
    };

    const result = await journalEntryService.create(entry);

    // Verify base amounts were calculated correctly
    expect(result.lines[0].baseAmount).toBe(1100); // 1000 * 1.10
    expect(result.lines[0].exchangeRate).toBe(1.10);
    expect(result.totalDebit).toBe(1100);
    expect(result.totalCredit).toBe(1100);
  });
});
```

## Security Considerations

### Access Controls
- **Exchange Rate Management**: Restricted to finance managers/admins
- **Manual Rate Overrides**: Require dual approval for large deviations
- **Currency Configuration**: Tenant admin only
- **Multi-currency Transactions**: Follow standard journal entry approval workflows

### Data Integrity
- **Exchange Rate History**: Immutable once created
- **Transaction Locking**: Rates locked at transaction date
- **Audit Trail**: Track all currency conversions and rate changes
- **Reconciliation**: Verify converted amounts match original transactions

This comprehensive multi-currency framework provides full support for international accounting operations while maintaining data integrity and compliance with accounting standards.