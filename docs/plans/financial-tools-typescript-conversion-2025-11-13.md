# Financial Tools TypeScript Conversion Plan

**Date**: 2025-11-13
**Status**: Planning
**Owner**: Dexter Team

## Executive Summary

This document provides a comprehensive plan for converting 14 financial data retrieval tools from Python (using LangChain's `@tool` decorator and Pydantic) to TypeScript using the Vercel AI SDK's `tool()` function pattern with Zod schemas.

**Key Goals**:
- Migrate all 14 financial tools to TypeScript
- Use Vercel AI SDK tool patterns
- Maintain API compatibility with Financial Datasets API
- Implement proper type safety with Zod
- Create reusable patterns for future tool development

---

## API Client Architecture

### Current Python Implementation

The Python implementation uses a simple `call_api()` function in `/src/dexter/tools/finance/api.py`:

```python
def call_api(endpoint: str, params: dict) -> dict:
    """Helper function to call the Financial Datasets API."""
    base_url = "https://api.financialdatasets.ai"
    url = f"{base_url}{endpoint}"
    headers = {"x-api-key": financial_datasets_api_key}
    response = requests.get(url, params=params, headers=headers)
    response.raise_for_status()
    return response.json()
```

### Proposed TypeScript Implementation

Create a centralized API client at `/src/lib/financial-datasets-client.ts`:

```typescript
/**
 * Financial Datasets API Client
 *
 * Provides a centralized client for interacting with the Financial Datasets API.
 * Includes error handling, request validation, and response parsing.
 */

interface ApiConfig {
  baseUrl: string;
  apiKey: string;
  timeout?: number;
}

interface ApiError extends Error {
  status?: number;
  endpoint?: string;
  params?: Record<string, unknown>;
}

class FinancialDatasetsClient {
  private baseUrl: string;
  private apiKey: string;
  private timeout: number;

  constructor(config?: Partial<ApiConfig>) {
    this.baseUrl = config?.baseUrl || 'https://api.financialdatasets.ai';
    this.apiKey = config?.apiKey || process.env.FINANCIAL_DATASETS_API_KEY || '';
    this.timeout = config?.timeout || 30000;

    if (!this.apiKey) {
      throw new Error('FINANCIAL_DATASETS_API_KEY environment variable is required');
    }
  }

  /**
   * Makes a GET request to the Financial Datasets API
   * @param endpoint - API endpoint (e.g., '/financials/income-statements/')
   * @param params - Query parameters
   * @returns Parsed JSON response
   */
  async callApi<T = unknown>(
    endpoint: string,
    params: Record<string, unknown> = {}
  ): Promise<T> {
    // Build URL with query parameters
    const url = new URL(endpoint, this.baseUrl);

    // Filter out undefined/null values and convert to strings
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        // Handle arrays (e.g., item parameter)
        if (Array.isArray(value)) {
          value.forEach(v => url.searchParams.append(key, String(v)));
        } else {
          url.searchParams.append(key, String(value));
        }
      }
    });

    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), this.timeout);

      const response = await fetch(url.toString(), {
        method: 'GET',
        headers: {
          'x-api-key': this.apiKey,
          'Content-Type': 'application/json',
        },
        signal: controller.signal,
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        const error: ApiError = new Error(
          `API request failed: ${response.status} ${response.statusText}`
        );
        error.status = response.status;
        error.endpoint = endpoint;
        error.params = params;
        throw error;
      }

      const data = await response.json();
      return data as T;
    } catch (error) {
      if (error instanceof Error) {
        if (error.name === 'AbortError') {
          throw new Error(`Request timeout after ${this.timeout}ms`);
        }
        throw error;
      }
      throw new Error('Unknown error occurred during API request');
    }
  }
}

// Export singleton instance
export const financialDatasetsClient = new FinancialDatasetsClient();

// Export class for testing/custom instances
export { FinancialDatasetsClient };
export type { ApiConfig, ApiError };
```

**Key Design Decisions**:
1. **Singleton Pattern**: Export a pre-configured instance for convenience
2. **Type Safety**: Generic `callApi<T>()` method for typed responses
3. **Error Handling**: Custom error types with request context
4. **Array Parameters**: Special handling for array query params (needed for SEC filing items)
5. **Timeout Support**: Configurable request timeouts with AbortController
6. **Environment Variables**: Automatic loading from `FINANCIAL_DATASETS_API_KEY`

---

## Tool Pattern

### Vercel AI SDK Tool Structure

Based on the AI SDK documentation, each tool follows this pattern:

```typescript
import { tool } from 'ai';
import { z } from 'zod';

const myTool = tool({
  description: 'Clear description for the LLM',
  inputSchema: z.object({
    param: z.string().describe('Parameter description'),
  }),
  execute: async ({ param }) => {
    // Implementation
    return result;
  },
});
```

### Standard Tool Template

All financial tools will follow this template at `/src/tools/finance/{category}.ts`:

```typescript
/**
 * {Category} Tools
 *
 * Tools for retrieving {category description} from Financial Datasets API.
 */

import { tool } from 'ai';
import { z } from 'zod';
import { financialDatasetsClient } from '@/lib/financial-datasets-client';

// Define Zod schemas for input validation
const toolNameInputSchema = z.object({
  ticker: z.string().describe('The stock ticker symbol (e.g., "AAPL" for Apple)'),
  // ... additional parameters
});

// Define tool
export const toolName = tool({
  description: 'Multi-line description explaining:\n' +
    '- What data the tool retrieves\n' +
    '- When to use it\n' +
    '- What insights it provides',
  inputSchema: toolNameInputSchema,
  execute: async (input) => {
    // Type-safe input from Zod schema
    const { ticker, ...otherParams } = input;

    // Build API parameters
    const params = {
      ticker: ticker.toUpperCase(),
      ...otherParams,
    };

    // Call API with typed response
    const response = await financialDatasetsClient.callApi<ResponseType>(
      '/endpoint/',
      params
    );

    // Extract and return relevant data
    return response.key_name || {};
  },
});

// Export type for tool result
export type ToolNameResult = z.infer<typeof toolNameInputSchema>;
```

### Key Conversions

| Python Pattern | TypeScript Pattern |
|----------------|-------------------|
| `@tool(args_schema=InputClass)` | `tool({ inputSchema: zodSchema })` |
| `class Input(BaseModel)` | `z.object({ ... })` |
| `Field(description="...")` | `z.string().describe("...")` |
| `Literal["a", "b"]` | `z.enum(["a", "b"])` |
| `Optional[str]` | `z.string().optional()` |
| `Field(default=10)` | `z.number().default(10)` |
| `list[str]` | `z.array(z.string())` |
| `requests.get()` | `fetch()` |

---

## Representative Examples

### Example 1: Simple Tool (get_price_snapshot)

**Python Version**:
```python
class PriceSnapshotInput(BaseModel):
    ticker: str = Field(..., description="The stock ticker symbol...")

@tool(args_schema=PriceSnapshotInput)
def get_price_snapshot(ticker: str) -> dict:
    """Fetches the most recent price snapshot..."""
    params = {"ticker": ticker}
    data = call_api("/prices/snapshot/", params)
    return data.get("snapshot", {})
```

**TypeScript Version** (`/src/tools/finance/prices.ts`):
```typescript
/**
 * Price Data Tools
 *
 * Tools for retrieving current and historical stock price data.
 */

import { tool } from 'ai';
import { z } from 'zod';
import { financialDatasetsClient } from '@/lib/financial-datasets-client';

// Response type definitions
interface PriceSnapshot {
  ticker: string;
  price: number;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
  timestamp: string;
  [key: string]: unknown;
}

interface PriceSnapshotResponse {
  snapshot: PriceSnapshot;
}

// Input schema
const priceSnapshotInputSchema = z.object({
  ticker: z
    .string()
    .describe('The stock ticker symbol to fetch the price snapshot for (e.g., "AAPL" for Apple)'),
});

// Tool definition
export const getPriceSnapshot = tool({
  description:
    'Fetches the most recent price snapshot for a specific stock, ' +
    'including the latest price, trading volume, and other open, high, low, and close price data. ' +
    'Use this when you need current market prices and trading activity.',
  inputSchema: priceSnapshotInputSchema,
  execute: async ({ ticker }) => {
    const params = { ticker: ticker.toUpperCase() };

    const response = await financialDatasetsClient.callApi<PriceSnapshotResponse>(
      '/prices/snapshot/',
      params
    );

    return response.snapshot || {};
  },
});

// Export types
export type PriceSnapshotInput = z.infer<typeof priceSnapshotInputSchema>;
```

### Example 2: Complex Tool with Multiple Parameters (get_income_statements)

**Python Version**:
```python
class FinancialStatementsInput(BaseModel):
    ticker: str = Field(description="...")
    period: Literal["annual", "quarterly", "ttm"] = Field(description="...")
    limit: int = Field(default=10, description="...")
    report_period_gt: Optional[str] = Field(default=None, description="...")
    # ... more date filters

@tool(args_schema=FinancialStatementsInput)
def get_income_statements(ticker: str, period: Literal[...], ...) -> dict:
    """Fetches income statements..."""
    params = _create_params(ticker, period, limit, ...)
    data = call_api("/financials/income-statements/", params)
    return data.get("income_statements", {})
```

**TypeScript Version** (`/src/tools/finance/fundamentals.ts`):
```typescript
/**
 * Financial Statements Tools
 *
 * Tools for retrieving income statements, balance sheets, and cash flow statements.
 */

import { tool } from 'ai';
import { z } from 'zod';
import { financialDatasetsClient } from '@/lib/financial-datasets-client';

// Response type definitions
interface IncomeStatement {
  ticker: string;
  period: string;
  report_period: string;
  revenue: number;
  cost_of_revenue: number;
  gross_profit: number;
  operating_expenses: number;
  net_income: number;
  [key: string]: unknown;
}

interface IncomeStatementsResponse {
  income_statements: IncomeStatement[];
}

// Shared schema for all financial statements
const periodEnum = z.enum(['annual', 'quarterly', 'ttm']).describe(
  "The reporting period: 'annual' for yearly, 'quarterly' for quarterly, 'ttm' for trailing twelve months"
);

const financialStatementsBaseSchema = z.object({
  ticker: z
    .string()
    .describe('The stock ticker symbol to fetch financial statements for (e.g., "AAPL" for Apple)'),
  period: periodEnum,
  limit: z
    .number()
    .int()
    .positive()
    .default(10)
    .describe('The number of past financial statements to retrieve'),
  report_period_gt: z
    .string()
    .optional()
    .describe('Filter for statements with report periods after this date (YYYY-MM-DD)'),
  report_period_gte: z
    .string()
    .optional()
    .describe('Filter for statements with report periods on or after this date (YYYY-MM-DD)'),
  report_period_lt: z
    .string()
    .optional()
    .describe('Filter for statements with report periods before this date (YYYY-MM-DD)'),
  report_period_lte: z
    .string()
    .optional()
    .describe('Filter for statements with report periods on or before this date (YYYY-MM-DD)'),
});

// Income Statements Tool
export const getIncomeStatements = tool({
  description:
    'Fetches a company\'s income statements, detailing its revenues, expenses, net income, etc. ' +
    'over a reporting period. Useful for evaluating a company\'s profitability and operational efficiency.\n\n' +
    'Key metrics included:\n' +
    '- Revenue (total sales)\n' +
    '- Cost of revenue\n' +
    '- Gross profit\n' +
    '- Operating expenses\n' +
    '- Net income (bottom line)\n' +
    '- Earnings per share (EPS)',
  inputSchema: financialStatementsBaseSchema,
  execute: async (input) => {
    const { ticker, period, limit, ...dateFilters } = input;

    // Build params object, filtering out undefined values
    const params: Record<string, unknown> = {
      ticker: ticker.toUpperCase(),
      period,
      limit,
    };

    // Add optional date filters
    if (dateFilters.report_period_gt) params.report_period_gt = dateFilters.report_period_gt;
    if (dateFilters.report_period_gte) params.report_period_gte = dateFilters.report_period_gte;
    if (dateFilters.report_period_lt) params.report_period_lt = dateFilters.report_period_lt;
    if (dateFilters.report_period_lte) params.report_period_lte = dateFilters.report_period_lte;

    const response = await financialDatasetsClient.callApi<IncomeStatementsResponse>(
      '/financials/income-statements/',
      params
    );

    return response.income_statements || [];
  },
});

// Balance Sheets Tool
export const getBalanceSheets = tool({
  description:
    'Retrieves a company\'s balance sheets, providing a snapshot of its assets, liabilities, ' +
    'shareholders\' equity, etc. at a specific point in time. Useful for assessing a company\'s financial position.\n\n' +
    'Key metrics included:\n' +
    '- Total assets (current + non-current)\n' +
    '- Total liabilities (current + long-term)\n' +
    '- Shareholders\' equity\n' +
    '- Working capital\n' +
    '- Debt levels',
  inputSchema: financialStatementsBaseSchema,
  execute: async (input) => {
    const { ticker, period, limit, ...dateFilters } = input;

    const params: Record<string, unknown> = {
      ticker: ticker.toUpperCase(),
      period,
      limit,
    };

    if (dateFilters.report_period_gt) params.report_period_gt = dateFilters.report_period_gt;
    if (dateFilters.report_period_gte) params.report_period_gte = dateFilters.report_period_gte;
    if (dateFilters.report_period_lt) params.report_period_lt = dateFilters.report_period_lt;
    if (dateFilters.report_period_lte) params.report_period_lte = dateFilters.report_period_lte;

    const response = await financialDatasetsClient.callApi<{ balance_sheets: unknown[] }>(
      '/financials/balance-sheets/',
      params
    );

    return response.balance_sheets || [];
  },
});

// Cash Flow Statements Tool
export const getCashFlowStatements = tool({
  description:
    'Retrieves a company\'s cash flow statements, showing how cash is generated and used across ' +
    'operating, investing, and financing activities. Useful for understanding a company\'s liquidity and solvency.\n\n' +
    'Key sections:\n' +
    '- Operating activities (cash from business operations)\n' +
    '- Investing activities (capital expenditures, acquisitions)\n' +
    '- Financing activities (debt, equity, dividends)',
  inputSchema: financialStatementsBaseSchema,
  execute: async (input) => {
    const { ticker, period, limit, ...dateFilters } = input;

    const params: Record<string, unknown> = {
      ticker: ticker.toUpperCase(),
      period,
      limit,
    };

    if (dateFilters.report_period_gt) params.report_period_gt = dateFilters.report_period_gt;
    if (dateFilters.report_period_gte) params.report_period_gte = dateFilters.report_period_gte;
    if (dateFilters.report_period_lt) params.report_period_lt = dateFilters.report_period_lt;
    if (dateFilters.report_period_lte) params.report_period_lte = dateFilters.report_period_lte;

    const response = await financialDatasetsClient.callApi<{ cash_flow_statements: unknown[] }>(
      '/financials/cash-flow-statements/',
      params
    );

    return response.cash_flow_statements || [];
  },
});

// Export types
export type FinancialStatementsInput = z.infer<typeof financialStatementsBaseSchema>;
```

### Example 3: Tool with Constants and Complex Logic (get_10K_filing_items)

**Python Version**:
```python
from dexter.tools.finance.constants import ITEMS_10K_MAP, format_items_description

class Filing10KItemsInput(BaseModel):
    ticker: str = Field(description="...")
    year: int = Field(description="...")
    item: Optional[list[str]] = Field(
        default=None,
        description=f"Optional list... Valid items:\n{format_items_description(ITEMS_10K_MAP)}"
    )

@tool(args_schema=Filing10KItemsInput)
def get_10K_filing_items(ticker: str, year: int, item: list[str] | None = None) -> dict:
    """Retrieves sections from 10-K..."""
    params = {"ticker": ticker.upper(), "filing_type": "10-K", "year": year}
    if item is not None:
        params["item"] = item
    data = call_api("/filings/items/", params)
    return data
```

**TypeScript Version** (`/src/tools/finance/filings.ts`):
```typescript
/**
 * SEC Filings Tools
 *
 * Tools for retrieving SEC filing metadata and content (10-K, 10-Q, 8-K).
 */

import { tool } from 'ai';
import { z } from 'zod';
import { financialDatasetsClient } from '@/lib/financial-datasets-client';
import {
  ITEMS_10K_MAP,
  ITEMS_10Q_MAP,
  ITEMS_8K_MAP,
  formatItemsDescription
} from '@/lib/constants/sec-filings';

// Response type definitions
interface FilingItem {
  number: string;
  title: string;
  text: string;
}

interface FilingItemsResponse {
  resource: string;
  ticker: string;
  cik: string;
  filing_type: string;
  accession_number: string;
  year?: number;
  quarter?: number;
  items: FilingItem[];
}

// 10-K Filing Items Tool
const filing10KItemsInputSchema = z.object({
  ticker: z
    .string()
    .describe('The stock ticker symbol (e.g., "AAPL" for Apple)'),
  year: z
    .number()
    .int()
    .min(1990)
    .max(new Date().getFullYear())
    .describe('The year of the 10-K filing (e.g., 2023)'),
  item: z
    .array(z.string())
    .optional()
    .describe(
      `Optional list of specific items to retrieve from the 10-K. Valid items are:\n${formatItemsDescription(ITEMS_10K_MAP)}\n\nIf not specified, all available items will be returned.`
    ),
});

export const get10KFilingItems = tool({
  description:
    'Retrieves specific sections (items) from a company\'s 10-K annual report. ' +
    '10-K filings provide comprehensive annual information about a company\'s business, risks, and financial performance.\n\n' +
    'Common sections include:\n' +
    '- Item-1: Business overview and operations\n' +
    '- Item-1A: Risk Factors (key risks facing the company)\n' +
    '- Item-7: Management\'s Discussion and Analysis (MD&A)\n' +
    '- Item-8: Financial Statements and Supplementary Data\n\n' +
    'Use this to extract detailed information from specific sections of a 10-K filing. ' +
    'The optional "item" parameter allows filtering for specific sections.',
  inputSchema: filing10KItemsInputSchema,
  execute: async ({ ticker, year, item }) => {
    const params: Record<string, unknown> = {
      ticker: ticker.toUpperCase(),
      filing_type: '10-K',
      year,
    };

    // Add item filter if provided
    if (item && item.length > 0) {
      params.item = item;
    }

    const response = await financialDatasetsClient.callApi<FilingItemsResponse>(
      '/filings/items/',
      params
    );

    return response;
  },
});

// 10-Q Filing Items Tool
const filing10QItemsInputSchema = z.object({
  ticker: z
    .string()
    .describe('The stock ticker symbol (e.g., "AAPL" for Apple)'),
  year: z
    .number()
    .int()
    .min(1990)
    .max(new Date().getFullYear())
    .describe('The year of the 10-Q filing (e.g., 2023)'),
  quarter: z
    .number()
    .int()
    .min(1)
    .max(4)
    .describe('The quarter of the 10-Q filing (1, 2, 3, or 4)'),
  item: z
    .array(z.string())
    .optional()
    .describe(
      `Optional list of specific items to retrieve from the 10-Q. Valid items are:\n${formatItemsDescription(ITEMS_10Q_MAP)}\n\nIf not specified, all available items will be returned.`
    ),
});

export const get10QFilingItems = tool({
  description:
    'Retrieves specific sections (items) from a company\'s 10-Q quarterly report. ' +
    '10-Q filings provide quarterly updates on financial performance and operations.\n\n' +
    'Common sections include:\n' +
    '- Item-1: Financial Statements (quarterly results)\n' +
    '- Item-2: Management\'s Discussion and Analysis (MD&A)\n' +
    '- Item-3: Quantitative and Qualitative Disclosures About Market Risk\n' +
    '- Item-4: Controls and Procedures\n\n' +
    'Use this to extract quarterly information and interim financial results.',
  inputSchema: filing10QItemsInputSchema,
  execute: async ({ ticker, year, quarter, item }) => {
    const params: Record<string, unknown> = {
      ticker: ticker.toUpperCase(),
      filing_type: '10-Q',
      year,
      quarter,
    };

    if (item && item.length > 0) {
      params.item = item;
    }

    const response = await financialDatasetsClient.callApi<FilingItemsResponse>(
      '/filings/items/',
      params
    );

    return response;
  },
});

// 8-K Filing Items Tool
const filing8KItemsInputSchema = z.object({
  ticker: z
    .string()
    .describe('The stock ticker symbol (e.g., "AAPL" for Apple)'),
  accession_number: z
    .string()
    .describe(
      'The SEC accession number for the 8-K filing (e.g., "0000320193-24-000123"). ' +
      'This can be retrieved from the getFilings tool.'
    ),
});

export const get8KFilingItems = tool({
  description:
    'Retrieves specific sections (items) from a company\'s 8-K current report. ' +
    '8-K filings report material events such as acquisitions, financial results, management changes, ' +
    'and other significant corporate events.\n\n' +
    'Common 8-K items include:\n' +
    '- Item-1.01: Entry into a Material Definitive Agreement\n' +
    '- Item-2.02: Results of Operations and Financial Condition\n' +
    '- Item-5.02: Departure/Election of Directors or Principal Officers\n' +
    '- Item-8.01: Other Events\n\n' +
    'The accession_number parameter can be retrieved using the getFilings tool by filtering for 8-K filings.',
  inputSchema: filing8KItemsInputSchema,
  execute: async ({ ticker, accession_number }) => {
    const params: Record<string, unknown> = {
      ticker: ticker.toUpperCase(),
      filing_type: '8-K',
      accession_number,
    };

    const response = await financialDatasetsClient.callApi<FilingItemsResponse>(
      '/filings/items/',
      params
    );

    return response;
  },
});

// Export types
export type Filing10KItemsInput = z.infer<typeof filing10KItemsInputSchema>;
export type Filing10QItemsInput = z.infer<typeof filing10QItemsInputSchema>;
export type Filing8KItemsInput = z.infer<typeof filing8KItemsInputSchema>;
```

---

## Tool Registry

### Registry Pattern

Create a central registry at `/src/tools/finance/index.ts`:

```typescript
/**
 * Financial Tools Registry
 *
 * Central export point for all financial data retrieval tools.
 * Organizes tools by category and provides type-safe access.
 */

// Financial Statements
export {
  getIncomeStatements,
  getBalanceSheets,
  getCashFlowStatements,
  type FinancialStatementsInput,
} from './fundamentals';

// SEC Filings
export {
  getFilings,
  get10KFilingItems,
  get10QFilingItems,
  get8KFilingItems,
  type FilingsInput,
  type Filing10KItemsInput,
  type Filing10QItemsInput,
  type Filing8KItemsInput,
} from './filings';

// Market Data
export {
  getPriceSnapshot,
  getPrices,
  type PriceSnapshotInput,
  type PricesInput,
} from './prices';

// Financial Metrics
export {
  getFinancialMetricsSnapshot,
  getFinancialMetrics,
  type FinancialMetricsSnapshotInput,
  type FinancialMetricsInput,
} from './metrics';

// Analysis
export {
  getAnalystEstimates,
  getSegmentedRevenues,
  type AnalystEstimatesInput,
  type SegmentedRevenuesInput,
} from './analysis';

// News
export {
  getNews,
  type NewsInput,
} from './news';

/**
 * All financial tools as a single object
 * Use this for registering all tools with AI SDK at once
 */
export const financialTools = {
  // Financial Statements
  getIncomeStatements,
  getBalanceSheets,
  getCashFlowStatements,

  // SEC Filings
  getFilings,
  get10KFilingItems,
  get10QFilingItems,
  get8KFilingItems,

  // Market Data
  getPriceSnapshot,
  getPrices,

  // Financial Metrics
  getFinancialMetricsSnapshot,
  getFinancialMetrics,

  // Analysis
  getAnalystEstimates,
  getSegmentedRevenues,

  // News
  getNews,
};

/**
 * Tool categories for selective registration
 */
export const toolCategories = {
  statements: {
    getIncomeStatements,
    getBalanceSheets,
    getCashFlowStatements,
  },
  filings: {
    getFilings,
    get10KFilingItems,
    get10QFilingItems,
    get8KFilingItems,
  },
  market: {
    getPriceSnapshot,
    getPrices,
  },
  metrics: {
    getFinancialMetricsSnapshot,
    getFinancialMetrics,
  },
  analysis: {
    getAnalystEstimates,
    getSegmentedRevenues,
  },
  news: {
    getNews,
  },
};

// Type for the complete tool set
export type FinancialToolSet = typeof financialTools;
```

### Usage Example

```typescript
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { financialTools, toolCategories } from '@/tools/finance';

// Use all tools
const result = await generateText({
  model: openai('gpt-4o'),
  tools: financialTools,
  prompt: 'Analyze Apple\'s latest financial performance',
});

// Use specific categories
const result2 = await generateText({
  model: openai('gpt-4o'),
  tools: {
    ...toolCategories.statements,
    ...toolCategories.metrics,
  },
  prompt: 'Compare revenue and profitability for AAPL',
});
```

---

## Constants & Configuration

### SEC Filing Mappings

Create `/src/lib/constants/sec-filings.ts`:

```typescript
/**
 * SEC Filing Constants
 *
 * Mappings for SEC filing item numbers to their descriptions,
 * used across various filing tools (10-K, 10-Q, 8-K).
 */

/**
 * 10-K Annual Report Items
 *
 * Source: SEC Form 10-K instructions
 */
export const ITEMS_10K_MAP = {
  'Item-1': 'Business',
  'Item-1A': 'Risk Factors',
  'Item-1B': 'Unresolved Staff Comments',
  'Item-2': 'Properties',
  'Item-3': 'Legal Proceedings',
  'Item-4': 'Mine Safety Disclosures',
  'Item-5': 'Market for Registrant\'s Common Equity, Related Stockholder Matters and Issuer Purchases of Equity Securities',
  'Item-6': '[Reserved]',
  'Item-7': 'Management\'s Discussion and Analysis of Financial Condition and Results of Operations',
  'Item-7A': 'Quantitative and Qualitative Disclosures About Market Risk',
  'Item-8': 'Financial Statements and Supplementary Data',
  'Item-9': 'Changes in and Disagreements With Accountants on Accounting and Financial Disclosure',
  'Item-9A': 'Controls and Procedures',
  'Item-9B': 'Other Information',
  'Item-10': 'Directors, Executive Officers and Corporate Governance',
  'Item-11': 'Executive Compensation',
  'Item-12': 'Security Ownership of Certain Beneficial Owners and Management and Related Stockholder Matters',
  'Item-13': 'Certain Relationships and Related Transactions, and Director Independence',
  'Item-14': 'Principal Accounting Fees and Services',
  'Item-15': 'Exhibits, Financial Statement Schedules',
  'Item-16': 'Form 10-K Summary',
} as const;

/**
 * 10-Q Quarterly Report Items
 *
 * Source: SEC Form 10-Q instructions
 */
export const ITEMS_10Q_MAP = {
  'Item-1': 'Financial Statements',
  'Item-2': 'Management\'s Discussion and Analysis of Financial Condition and Results of Operations',
  'Item-3': 'Quantitative and Qualitative Disclosures About Market Risk',
  'Item-4': 'Controls and Procedures',
} as const;

/**
 * 8-K Current Report Items
 *
 * Source: SEC Form 8-K instructions
 */
export const ITEMS_8K_MAP = {
  'Item-1.01': 'Entry into a Material Definitive Agreement',
  'Item-1.02': 'Termination of a Material Definitive Agreement',
  'Item-1.03': 'Bankruptcy or Receivership',
  'Item-1.04': 'Mine Safety - Reporting of Shutdowns and Patterns of Violations',
  'Item-2.01': 'Completion of Acquisition or Disposition of Assets',
  'Item-2.02': 'Results of Operations and Financial Condition',
  'Item-2.03': 'Creation of a Direct Financial Obligation or an Obligation under an Off-Balance Sheet Arrangement',
  'Item-2.04': 'Triggering Events That Accelerate or Increase a Direct Financial Obligation',
  'Item-2.05': 'Costs Associated with Exit or Disposal Activities',
  'Item-2.06': 'Material Impairments',
  'Item-3.01': 'Notice of Delisting or Failure to Satisfy a Continued Listing Rule or Standard',
  'Item-3.02': 'Unregistered Sales of Equity Securities',
  'Item-3.03': 'Material Modification to Rights of Security Holders',
  'Item-4.01': 'Changes in Registrant\'s Certifying Accountant',
  'Item-4.02': 'Non-Reliance on Previously Issued Financial Statements or a Related Audit Report',
  'Item-5.01': 'Changes in Control of Registrant',
  'Item-5.02': 'Departure of Directors or Certain Officers; Election of Directors; Appointment of Certain Officers',
  'Item-5.03': 'Amendments to Articles of Incorporation or Bylaws; Change in Fiscal Year',
  'Item-5.04': 'Temporary Suspension of Trading Under Registrant\'s Employee Benefit Plans',
  'Item-5.05': 'Amendment to Registrant\'s Code of Ethics, or Waiver of a Provision of the Code of Ethics',
  'Item-5.06': 'Change in Shell Company Status',
  'Item-5.07': 'Submission of Matters to a Vote of Security Holders',
  'Item-5.08': 'Shareholder Director Nominations',
  'Item-6.01': 'ABS Informational and Computational Material',
  'Item-6.02': 'Change of Servicer or Trustee',
  'Item-6.03': 'Change in Credit Enhancement or Other External Support',
  'Item-6.04': 'Failure to Make a Required Distribution',
  'Item-6.05': 'Securities Act Updating Disclosure',
  'Item-7.01': 'Regulation FD Disclosure',
  'Item-8.01': 'Other Events',
  'Item-9.01': 'Financial Statements and Exhibits',
} as const;

/**
 * Helper function to format item mappings into readable descriptions
 * for tool schemas.
 *
 * @param itemsMap - Dictionary mapping item codes to their descriptions
 * @returns Formatted string with each item on a new line
 *
 * @example
 * ```typescript
 * formatItemsDescription(ITEMS_10K_MAP)
 * // Returns:
 * // "  - Item-1: Business
 * //    - Item-1A: Risk Factors
 * //    ..."
 * ```
 */
export function formatItemsDescription(
  itemsMap: Record<string, string>
): string {
  return Object.entries(itemsMap)
    .map(([item, description]) => `  - ${item}: ${description}`)
    .join('\n');
}

// Export arrays for backwards compatibility
export const ITEMS_10K = Object.keys(ITEMS_10K_MAP);
export const ITEMS_10Q = Object.keys(ITEMS_10Q_MAP);
export const ITEMS_8K = Object.keys(ITEMS_8K_MAP);

// Type exports
export type Item10K = keyof typeof ITEMS_10K_MAP;
export type Item10Q = keyof typeof ITEMS_10Q_MAP;
export type Item8K = keyof typeof ITEMS_8K_MAP;
```

---

## Migration Checklist

### Phase 1: Foundation (Week 1)

- [ ] **Setup TypeScript Project Structure**
  - [ ] Create `/src/lib/` directory
  - [ ] Create `/src/tools/finance/` directory
  - [ ] Setup `tsconfig.json` with strict mode
  - [ ] Install dependencies: `npm install ai zod`

- [ ] **Implement Core Infrastructure**
  - [ ] Create `financial-datasets-client.ts`
  - [ ] Create `sec-filings.ts` constants
  - [ ] Add unit tests for API client
  - [ ] Setup environment variables

- [ ] **Create Documentation**
  - [ ] Tool development guide
  - [ ] API client usage examples
  - [ ] Zod schema patterns

### Phase 2: Financial Statements (Week 2)

- [ ] **Fundamentals Tools** (`fundamentals.ts`)
  - [ ] Convert `get_income_statements`
  - [ ] Convert `get_balance_sheets`
  - [ ] Convert `get_cash_flow_statements`
  - [ ] Add response type definitions
  - [ ] Write unit tests
  - [ ] Integration tests with API

### Phase 3: SEC Filings (Week 2-3)

- [ ] **Filings Tools** (`filings.ts`)
  - [ ] Convert `get_filings`
  - [ ] Convert `get_10K_filing_items`
  - [ ] Convert `get_10Q_filing_items`
  - [ ] Convert `get_8K_filing_items`
  - [ ] Add response type definitions
  - [ ] Write unit tests
  - [ ] Integration tests with API

### Phase 4: Market Data (Week 3)

- [ ] **Prices Tools** (`prices.ts`)
  - [ ] Convert `get_price_snapshot`
  - [ ] Convert `get_prices`
  - [ ] Add response type definitions
  - [ ] Write unit tests
  - [ ] Integration tests with API

### Phase 5: Metrics & Analysis (Week 3-4)

- [ ] **Metrics Tools** (`metrics.ts`)
  - [ ] Convert `get_financial_metrics_snapshot`
  - [ ] Convert `get_financial_metrics`
  - [ ] Add response type definitions
  - [ ] Write unit tests

- [ ] **Analysis Tools** (`analysis.ts`)
  - [ ] Convert `get_analyst_estimates`
  - [ ] Convert `get_segmented_revenues`
  - [ ] Add response type definitions
  - [ ] Write unit tests

### Phase 6: News & Registry (Week 4)

- [ ] **News Tools** (`news.ts`)
  - [ ] Convert `get_news`
  - [ ] Add response type definitions
  - [ ] Write unit tests

- [ ] **Tool Registry** (`index.ts`)
  - [ ] Create centralized exports
  - [ ] Organize by category
  - [ ] Add usage documentation
  - [ ] Create integration examples

### Phase 7: Testing & Documentation (Week 5)

- [ ] **End-to-End Testing**
  - [ ] Test all tools with real API
  - [ ] Test tool combinations
  - [ ] Test error scenarios
  - [ ] Performance testing

- [ ] **Documentation**
  - [ ] API reference docs
  - [ ] Usage examples
  - [ ] Migration guide
  - [ ] Troubleshooting guide

### Phase 8: Deployment (Week 5)

- [ ] **Production Readiness**
  - [ ] Code review
  - [ ] Security audit
  - [ ] Performance optimization
  - [ ] Deprecate Python tools
  - [ ] Update agent configuration

---

## Challenges & Solutions

### Challenge 1: Python `requests` → TypeScript `fetch`

**Issue**: Python's `requests` library has simpler error handling than `fetch`.

**Solution**:
- Use `AbortController` for timeouts
- Check `response.ok` before parsing JSON
- Create custom `ApiError` type with request context
- Wrap in try-catch for network errors

### Challenge 2: Pydantic `BaseModel` → Zod Schemas

**Issue**: Pydantic validation happens at class instantiation; Zod needs explicit parsing.

**Solution**:
- AI SDK automatically validates input with Zod schema
- No manual parsing needed in `execute` function
- Type inference works automatically from schema

### Challenge 3: Date Handling

**Issue**: Python's `datetime` vs TypeScript `Date` and ISO strings.

**Solution**:
- Use ISO 8601 strings (YYYY-MM-DD) for API params
- Add Zod validation: `z.string().regex(/^\d{4}-\d{2}-\d{2}$/)`
- Document expected format in descriptions

### Challenge 4: Array Query Parameters

**Issue**: Some tools (like filing items) accept array parameters that need special URL encoding.

**Solution**:
- API client handles arrays specially
- Calls `searchParams.append()` multiple times for arrays
- Maintains compatibility with Python's `requests` behavior

### Challenge 5: Optional Parameters

**Issue**: Python uses `None` for optional params; TypeScript uses `undefined`.

**Solution**:
- Use Zod's `.optional()` method
- Filter out `undefined` values before API call
- Use `if (value !== undefined && value !== null)` checks

### Challenge 6: Type Safety for API Responses

**Issue**: API responses are untyped JSON.

**Solution**:
- Define TypeScript interfaces for all response types
- Use generics: `callApi<ResponseType>(endpoint, params)`
- Runtime validation could be added with Zod if needed
- Document expected response structure

---

## Best Practices

### 1. Tool Descriptions

Write comprehensive descriptions that help the LLM understand:
- **What** data the tool retrieves
- **When** to use it (use cases)
- **What** insights it provides
- **Key metrics** included

Example:
```typescript
description:
  'Fetches a company\'s income statements, detailing its revenues, expenses, net income, etc. ' +
  'over a reporting period. Useful for evaluating a company\'s profitability and operational efficiency.\n\n' +
  'Key metrics included:\n' +
  '- Revenue (total sales)\n' +
  '- Cost of revenue\n' +
  '- Gross profit\n' +
  '- Operating expenses\n' +
  '- Net income (bottom line)\n' +
  '- Earnings per share (EPS)',
```

### 2. Parameter Descriptions

Make parameter descriptions clear and actionable:
```typescript
ticker: z.string().describe(
  'The stock ticker symbol to fetch data for (e.g., "AAPL" for Apple)'
),
```

### 3. Input Validation

Use Zod's full validation capabilities:
```typescript
year: z.number()
  .int()
  .min(1990)
  .max(new Date().getFullYear())
  .describe('The year of the filing'),
```

### 4. Error Handling

Provide context in error messages:
```typescript
catch (error) {
  if (error instanceof Error) {
    throw new Error(
      `Failed to fetch income statements for ${ticker}: ${error.message}`
    );
  }
  throw error;
}
```

### 5. Response Processing

Extract and return only relevant data:
```typescript
const response = await financialDatasetsClient.callApi<ResponseType>(
  '/endpoint/',
  params
);

// Return the relevant nested data
return response.income_statements || [];
```

### 6. Type Exports

Always export types for tool inputs:
```typescript
export type IncomeStatementsInput = z.infer<typeof incomeStatementsInputSchema>;
```

### 7. Reusable Schemas

Create shared schemas for common patterns:
```typescript
const periodEnum = z.enum(['annual', 'quarterly', 'ttm']);

const baseFinancialSchema = z.object({
  ticker: z.string(),
  period: periodEnum,
  limit: z.number().default(10),
});
```

---

## Testing Strategy

### Unit Tests

Test individual components in isolation:

```typescript
// Test API client
describe('FinancialDatasetsClient', () => {
  it('should make GET requests with correct headers', async () => {
    // Mock fetch
    // Test client behavior
  });

  it('should handle API errors gracefully', async () => {
    // Test error cases
  });
});

// Test tool schemas
describe('Income Statements Schema', () => {
  it('should validate valid input', () => {
    const result = financialStatementsBaseSchema.safeParse({
      ticker: 'AAPL',
      period: 'annual',
      limit: 5,
    });
    expect(result.success).toBe(true);
  });

  it('should reject invalid period', () => {
    const result = financialStatementsBaseSchema.safeParse({
      ticker: 'AAPL',
      period: 'invalid',
    });
    expect(result.success).toBe(false);
  });
});
```

### Integration Tests

Test tools with real API:

```typescript
describe('Get Income Statements Integration', () => {
  it('should fetch real income statements', async () => {
    const result = await getIncomeStatements.execute({
      ticker: 'AAPL',
      period: 'annual',
      limit: 1,
    });

    expect(result).toBeDefined();
    expect(Array.isArray(result)).toBe(true);
  });
});
```

### E2E Tests

Test tools with AI SDK:

```typescript
describe('Financial Tools E2E', () => {
  it('should work with generateText', async () => {
    const result = await generateText({
      model: openai('gpt-4o'),
      tools: { getIncomeStatements },
      prompt: 'Get Apple\'s latest annual income statement',
    });

    expect(result.toolCalls).toHaveLength(1);
  });
});
```

---

## Dependencies

### Required Packages

```json
{
  "dependencies": {
    "ai": "^4.0.0",
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "vitest": "^1.0.0"
  }
}
```

### Environment Variables

```bash
# .env
FINANCIAL_DATASETS_API_KEY=your_api_key_here

# Optional: Override base URL for testing
FINANCIAL_DATASETS_API_BASE_URL=https://api.financialdatasets.ai
```

---

## Success Criteria

### Functional Requirements

- [ ] All 14 tools converted and working
- [ ] API compatibility maintained
- [ ] Type safety enforced with Zod
- [ ] Error handling comprehensive
- [ ] Tests passing (unit + integration)

### Performance Requirements

- [ ] API response times < 2s average
- [ ] Memory usage reasonable
- [ ] No memory leaks

### Quality Requirements

- [ ] Code coverage > 80%
- [ ] TypeScript strict mode enabled
- [ ] ESLint rules passing
- [ ] Documentation complete

### User Experience Requirements

- [ ] Tool descriptions clear and helpful
- [ ] Parameter validation provides good error messages
- [ ] Examples and usage guides available
- [ ] Migration from Python seamless

---

## Timeline

**Total Duration**: 5 weeks

- **Week 1**: Foundation and infrastructure
- **Week 2**: Financial statements and begin filings
- **Week 3**: Complete filings, add market data and metrics
- **Week 4**: Analysis, news, and registry
- **Week 5**: Testing, documentation, and deployment

---

## Next Steps

1. **Review this plan** with the team
2. **Setup project structure** and dependencies
3. **Start Phase 1**: Build foundation (API client, constants)
4. **Implement tools** following the representative examples
5. **Test thoroughly** with real API
6. **Document** as you go
7. **Deploy** and deprecate Python tools

---

## References

- [Vercel AI SDK Documentation](https://sdk.vercel.ai/docs)
- [Zod Documentation](https://zod.dev)
- [Financial Datasets API Docs](https://financialdatasets.ai/docs)
- [SEC Filing Forms Reference](https://www.sec.gov/forms)

---

**Document Version**: 1.0
**Last Updated**: 2025-11-13
**Status**: Planning Phase
