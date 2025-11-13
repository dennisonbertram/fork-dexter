# Testing Templates for Dexter TypeScript Conversion

**Date**: 2025-11-13
**Status**: Ready for Implementation
**Framework**: Evalite + Vitest + Autoevals

---

## Table of Contents

1. [Testing Strategy Overview](#testing-strategy-overview)
2. [Evalite Setup](#evalite-setup)
3. [Unit Testing Templates](#unit-testing-templates)
4. [Integration Testing Templates](#integration-testing-templates)
5. [Evaluation Testing Templates](#evaluation-testing-templates)
6. [Custom Scorers](#custom-scorers)
7. [Testing Checklist](#testing-checklist)

---

## Testing Strategy Overview

### Three Testing Layers

**1. Unit Tests** (Vitest)
- Individual tool functions
- Schema validation
- Utility functions
- Fast feedback loop

**2. Integration Tests** (Vitest)
- Agent orchestration
- Multi-tool workflows
- CLI interaction flows
- End-to-end scenarios

**3. Evaluation Tests** (Evalite)
- Financial research correctness
- Answer quality (LLM-as-judge)
- Consistency across model providers
- Regression testing against Python baseline

### Testing Philosophy

```
Unit Tests → Fast feedback, high coverage
Integration Tests → Real-world scenarios, workflow validation
Evaluations → Quality assurance, correctness scoring
```

---

## Evalite Setup

### Installation

```bash
# Install Evalite and dependencies
pnpm add -D evalite vitest autoevals

# Rebuild better-sqlite3 (common issue fix)
pnpm rebuild better-sqlite3
```

### Package.json Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:watch": "vitest --watch",
    "test:ui": "vitest --ui",
    "eval": "evalite",
    "eval:dev": "evalite watch",
    "eval:export": "evalite export",
    "test:all": "pnpm test && pnpm eval"
  }
}
```

### Project Structure

```
dexter-ts/
├── tests/
│   ├── unit/                     # Vitest unit tests
│   │   ├── tools/
│   │   ├── agent/
│   │   └── utils/
│   ├── integration/              # Vitest integration tests
│   │   ├── agent-workflow.test.ts
│   │   └── cli-interaction.test.ts
│   └── evals/                    # Evalite evaluations
│       ├── financial-research.eval.ts
│       ├── answer-quality.eval.ts
│       ├── tool-accuracy.eval.ts
│       └── scorers/
│           ├── correctness.ts
│           ├── factuality.ts
│           └── financial-accuracy.ts
├── vitest.config.ts
└── evalite.config.ts
```

### Vitest Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'tests/',
        '**/*.test.ts',
        '**/*.eval.ts',
      ],
    },
    testTimeout: 30000, // 30s for LLM calls
    hookTimeout: 30000,
  },
});
```

### Evalite Configuration

```typescript
// evalite.config.ts
import { defineConfig } from 'evalite';

export default defineConfig({
  // Optional: customize evaluation settings
  outputPath: './eval-results',
  port: 3006,
});
```

---

## Unit Testing Templates

### Template 1: Tool Function Test

```typescript
// tests/unit/tools/fundamentals.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { getIncomeStatements } from '@/tools/finance/fundamentals';
import { FinancialDatasetsClient } from '@/tools/finance/api';

// Mock the API client
vi.mock('@/tools/finance/api');

describe('getIncomeStatements Tool', () => {
  let mockClient: any;

  beforeEach(() => {
    mockClient = {
      get: vi.fn(),
    };
    vi.mocked(FinancialDatasetsClient).mockImplementation(() => mockClient);
  });

  it('should fetch income statements for valid ticker', async () => {
    // Arrange
    const mockData = {
      statements: [
        {
          period: 'Q1 2024',
          revenue: 100000000,
          netIncome: 25000000,
        },
      ],
    };
    mockClient.get.mockResolvedValue(mockData);

    // Act
    const result = await getIncomeStatements.execute({
      ticker: 'AAPL',
      period: 'quarter',
      limit: 4,
    });

    // Assert
    expect(mockClient.get).toHaveBeenCalledWith(
      '/financials/income-statements',
      { ticker: 'AAPL', period: 'quarter', limit: 4 }
    );
    expect(result).toEqual(mockData);
  });

  it('should throw error for invalid ticker format', async () => {
    // Act & Assert
    await expect(
      getIncomeStatements.execute({
        ticker: 'invalid!@#',
        period: 'quarter',
        limit: 4,
      })
    ).rejects.toThrow('Invalid ticker format');
  });

  it('should handle API errors gracefully', async () => {
    // Arrange
    mockClient.get.mockRejectedValue(new Error('API rate limit exceeded'));

    // Act & Assert
    await expect(
      getIncomeStatements.execute({
        ticker: 'AAPL',
        period: 'quarter',
        limit: 4,
      })
    ).rejects.toThrow('API rate limit exceeded');
  });
});
```

### Template 2: Schema Validation Test

```typescript
// tests/unit/schemas.test.ts
import { describe, it, expect } from 'vitest';
import {
  TaskSchema,
  TaskListSchema,
  ToolCallSchema,
  AgentStateSchema,
} from '@/schemas';

describe('Zod Schemas', () => {
  describe('TaskSchema', () => {
    it('should validate correct task object', () => {
      const validTask = {
        id: 1,
        description: 'Get AAPL stock price',
        completed: false,
      };

      const result = TaskSchema.safeParse(validTask);
      expect(result.success).toBe(true);
      if (result.success) {
        expect(result.data).toEqual(validTask);
      }
    });

    it('should reject task with missing fields', () => {
      const invalidTask = {
        description: 'Get stock price',
        // missing id and completed
      };

      const result = TaskSchema.safeParse(invalidTask);
      expect(result.success).toBe(false);
    });

    it('should reject task with wrong types', () => {
      const invalidTask = {
        id: '1', // should be number
        description: 'Get stock price',
        completed: 'false', // should be boolean
      };

      const result = TaskSchema.safeParse(invalidTask);
      expect(result.success).toBe(false);
    });
  });

  describe('TaskListSchema', () => {
    it('should validate array of tasks', () => {
      const validTasks = {
        tasks: [
          { id: 1, description: 'Task 1', completed: false },
          { id: 2, description: 'Task 2', completed: true },
        ],
      };

      const result = TaskListSchema.safeParse(validTasks);
      expect(result.success).toBe(true);
    });
  });
});
```

### Template 3: Utility Function Test

```typescript
// tests/unit/utils/ui.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { UI, Colors, Spinner } from '@/utils/ui';

describe('UI Class', () => {
  let ui: UI;
  let consoleLogSpy: any;

  beforeEach(() => {
    ui = new UI();
    consoleLogSpy = vi.spyOn(console, 'log').mockImplementation(() => {});
  });

  afterEach(() => {
    consoleLogSpy.mockRestore();
  });

  describe('print', () => {
    it('should print message to console', () => {
      ui.print('Test message');
      expect(consoleLogSpy).toHaveBeenCalledWith('Test message');
    });

    it('should print colored message', () => {
      ui.printSuccess('Success message');
      expect(consoleLogSpy).toHaveBeenCalled();
    });
  });

  describe('spinner', () => {
    it('should start and stop spinner', async () => {
      ui.startSpinner('Loading...');
      expect(ui['spinner']).not.toBeNull();

      ui.stopSpinner('Done');
      expect(ui['spinner']).toBeNull();
    });

    it('should update spinner message', () => {
      ui.startSpinner('Loading...');
      ui.updateSpinner('Still loading...');
      ui.stopSpinner('Done');

      expect(ui['spinner']).toBeNull();
    });
  });
});
```

---

## Integration Testing Templates

### Template 4: Agent Workflow Test

```typescript
// tests/integration/agent-workflow.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { DexterAgent } from '@/agent';
import { anthropic } from '@ai-sdk/anthropic';

describe('Agent Workflow Integration', () => {
  let agent: DexterAgent;

  beforeAll(() => {
    agent = new DexterAgent({
      model: anthropic('claude-sonnet-4-5'),
      apiKey: process.env.ANTHROPIC_API_KEY!,
    });
  });

  it('should complete simple stock price query', async () => {
    const query = 'What is the current price of Apple stock?';
    const result = await agent.run(query);

    expect(result).toBeDefined();
    expect(result.length).toBeGreaterThan(0);
    expect(result.toLowerCase()).toContain('apple');
  }, 30000); // 30s timeout for LLM calls

  it('should execute multi-step research workflow', async () => {
    const query = 'Compare Tesla and Ford revenue for last quarter';
    const result = await agent.run(query);

    expect(result).toBeDefined();
    expect(result.toLowerCase()).toContain('tesla');
    expect(result.toLowerCase()).toContain('ford');
    expect(result.toLowerCase()).toMatch(/revenue|sales/);
  }, 60000); // 60s timeout for complex queries

  it('should handle tool execution errors gracefully', async () => {
    const query = 'Get financial data for INVALID_TICKER_12345';

    // Should not throw, but should return error message
    const result = await agent.run(query);
    expect(result).toBeDefined();
    expect(result.toLowerCase()).toMatch(/error|invalid|not found/);
  }, 30000);
});
```

### Template 5: Multi-Tool Coordination Test

```typescript
// tests/integration/multi-tool.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { DexterAgent } from '@/agent';
import { anthropic } from '@ai-sdk/anthropic';

describe('Multi-Tool Coordination', () => {
  let agent: DexterAgent;

  beforeAll(() => {
    agent = new DexterAgent({
      model: anthropic('claude-sonnet-4-5'),
    });
  });

  it('should coordinate income statement and metrics tools', async () => {
    const query = 'Get Apple income statement and P/E ratio';
    const result = await agent.run(query);

    expect(result).toBeDefined();
    // Should have called both tools
    expect(result.toLowerCase()).toMatch(/revenue|income/);
    expect(result.toLowerCase()).toMatch(/p\/e|pe ratio/);
  }, 45000);

  it('should retrieve filings and then extract specific sections', async () => {
    const query = 'Find Tesla latest 10-K and summarize business description';
    const result = await agent.run(query);

    expect(result).toBeDefined();
    expect(result.toLowerCase()).toContain('tesla');
    expect(result.toLowerCase()).toMatch(/10-k|business/);
  }, 60000);
});
```

---

## Evaluation Testing Templates

### Template 6: Financial Research Correctness Eval

```typescript
// tests/evals/financial-research.eval.ts
import { evalite } from 'evalite';
import { Factuality, ContextRecall } from 'autoevals';
import { DexterAgent } from '@/agent';
import { anthropic } from '@ai-sdk/anthropic';
import { financialCorrectnessScorer } from './scorers/correctness';

const agent = new DexterAgent({
  model: anthropic('claude-sonnet-4-5'),
});

evalite('Financial Research Correctness', {
  data: [
    {
      input: 'What was Apple revenue in Q4 2023?',
      expected: 'Apple reported revenue of $119.6 billion in Q4 2023 (fiscal Q1 2024).',
    },
    {
      input: 'What is Tesla P/E ratio?',
      expected: 'Tesla P/E ratio is approximately 60-80 (varies by market conditions).',
    },
    {
      input: 'When did Microsoft file their latest 10-K?',
      expected: 'Microsoft filed their latest 10-K in July 2024 for fiscal year ending June 30, 2024.',
    },
    {
      input: 'Compare Amazon and Google cloud revenue growth',
      expected: 'Amazon AWS grew ~13% YoY, Google Cloud grew ~28% YoY in recent quarters.',
    },
  ],
  task: async (input) => {
    const result = await agent.run(input);
    return result;
  },
  scorers: [
    Factuality, // Built-in factuality scorer from autoevals
    financialCorrectnessScorer, // Custom scorer
  ],
});
```

### Template 7: Answer Quality Eval (LLM-as-Judge)

```typescript
// tests/evals/answer-quality.eval.ts
import { evalite } from 'evalite';
import { DexterAgent } from '@/agent';
import { anthropic } from '@ai-sdk/anthropic';
import { answerQualityScorer } from './scorers/quality';

const agent = new DexterAgent({
  model: anthropic('claude-sonnet-4-5'),
});

evalite('Answer Quality Assessment', {
  data: [
    {
      input: 'Analyze Apple stock fundamentals',
      criteria: 'Should include revenue trends, profitability metrics, and forward guidance',
    },
    {
      input: 'What are the main risks for Tesla?',
      criteria: 'Should identify operational, financial, and market risks from SEC filings',
    },
    {
      input: 'Compare Microsoft and Google cloud businesses',
      criteria: 'Should compare revenue, growth rates, market share, and competitive positioning',
    },
  ],
  task: async (input) => {
    return await agent.run(input);
  },
  scorers: [answerQualityScorer],
});
```

### Template 8: Tool Accuracy Eval

```typescript
// tests/evals/tool-accuracy.eval.ts
import { evalite } from 'evalite';
import { Levenshtein } from 'autoevals';
import { getStockPrice, getIncomeStatements } from '@/tools/finance';

evalite('Stock Price Tool Accuracy', {
  data: [
    { input: { ticker: 'AAPL' }, expected: /^\d+\.\d{2}$/ },
    { input: { ticker: 'MSFT' }, expected: /^\d+\.\d{2}$/ },
    { input: { ticker: 'GOOGL' }, expected: /^\d+\.\d{2}$/ },
  ],
  task: async ({ ticker }) => {
    const result = await getStockPrice.execute({ ticker });
    return result.price.toString();
  },
  scorers: [
    {
      name: 'Price Format Validation',
      score: ({ output, expected }) => {
        const regex = expected as RegExp;
        return regex.test(output) ? 1 : 0;
      },
    },
  ],
});

evalite('Income Statement Tool Accuracy', {
  data: [
    {
      input: { ticker: 'AAPL', period: 'quarter', limit: 1 },
      expected: {
        hasRevenue: true,
        hasNetIncome: true,
        hasExpenses: true,
      },
    },
  ],
  task: async (input) => {
    const result = await getIncomeStatements.execute(input);
    return result;
  },
  scorers: [
    {
      name: 'Required Fields Present',
      score: ({ output, expected }) => {
        const data = output as any;
        const checks = expected as any;

        let score = 0;
        if (checks.hasRevenue && data.statements[0]?.revenue) score += 0.33;
        if (checks.hasNetIncome && data.statements[0]?.netIncome) score += 0.33;
        if (checks.hasExpenses && data.statements[0]?.expenses) score += 0.34;

        return score;
      },
    },
  ],
});
```

### Template 9: Cross-Model Consistency Eval

```typescript
// tests/evals/cross-model-consistency.eval.ts
import { evalite } from 'evalite';
import { DexterAgent } from '@/agent';
import { anthropic } from '@ai-sdk/anthropic';
import { openai } from '@ai-sdk/openai';

// Test same query across different models
evalite('Cross-Model Consistency', {
  data: [
    { input: 'What was Apple revenue last quarter?', provider: 'claude' },
    { input: 'What was Apple revenue last quarter?', provider: 'gpt4' },
  ],
  task: async ({ input, provider }) => {
    const model = provider === 'claude'
      ? anthropic('claude-sonnet-4-5')
      : openai('gpt-4o');

    const agent = new DexterAgent({ model });
    return await agent.run(input);
  },
  scorers: [
    {
      name: 'Consistency Check',
      score: ({ output }, { allOutputs }) => {
        // Compare if both models extracted similar numerical values
        const numbers = output.match(/\d+\.?\d*/g) || [];
        const allNumbers = allOutputs.map((o: string) =>
          o.match(/\d+\.?\d*/g) || []
        );

        // Simple consistency: both should extract similar values
        // More sophisticated comparison would parse currency/context
        return numbers.length > 0 && allNumbers.every(n => n.length > 0) ? 1 : 0;
      },
    },
  ],
});
```

---

## Custom Scorers

### Custom Scorer 1: Financial Correctness

```typescript
// tests/evals/scorers/correctness.ts
import { createScorer } from 'evalite';
import { z } from 'zod';
import { generateObject } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

export const financialCorrectnessScorer = createScorer({
  name: 'Financial Correctness',
  score: async ({ input, output, expected }) => {
    const result = await generateObject({
      model: anthropic('claude-sonnet-4-5'),
      schema: z.object({
        factuallyCorrect: z.boolean(),
        numericalAccuracy: z.number().min(0).max(1),
        sourceReliability: z.number().min(0).max(1),
        reasoning: z.string(),
      }),
      prompt: `
You are evaluating a financial research agent's answer.

Query: ${input}
Agent Answer: ${output}
Expected Answer: ${expected}

Evaluate the agent's answer on:
1. Factual correctness (boolean)
2. Numerical accuracy (0-1 score)
3. Source reliability (0-1 score)
4. Provide detailed reasoning

Scoring guidelines:
- Factual correctness: Are the core facts correct?
- Numerical accuracy: Are numbers within acceptable ranges? (±5% tolerance for estimates)
- Source reliability: Would this data come from reliable financial sources?
      `.trim(),
    });

    const { factuallyCorrect, numericalAccuracy, sourceReliability, reasoning } = result.object;

    const finalScore = factuallyCorrect
      ? (numericalAccuracy + sourceReliability) / 2
      : 0;

    return {
      score: finalScore,
      metadata: {
        factuallyCorrect,
        numericalAccuracy,
        sourceReliability,
        reasoning,
      },
    };
  },
});
```

### Custom Scorer 2: Answer Quality (LLM-as-Judge)

```typescript
// tests/evals/scorers/quality.ts
import { createScorer } from 'evalite';
import { z } from 'zod';
import { generateObject } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

export const answerQualityScorer = createScorer({
  name: 'Answer Quality',
  score: async ({ input, output, data }) => {
    const criteria = data.criteria;

    const result = await generateObject({
      model: anthropic('claude-sonnet-4-5'),
      schema: z.object({
        completeness: z.number().min(0).max(1),
        relevance: z.number().min(0).max(1),
        clarity: z.number().min(0).max(1),
        usefulness: z.number().min(0).max(1),
        reasoning: z.string(),
      }),
      prompt: `
You are evaluating the quality of a financial research agent's answer.

Query: ${input}
Answer: ${output}
Quality Criteria: ${criteria}

Rate the answer on a 0-1 scale for:
1. Completeness - Does it address all aspects of the query?
2. Relevance - Is the information directly relevant?
3. Clarity - Is it well-structured and easy to understand?
4. Usefulness - Would this be useful for making financial decisions?

Provide detailed reasoning for your scores.
      `.trim(),
    });

    const { completeness, relevance, clarity, usefulness, reasoning } = result.object;

    const finalScore = (completeness + relevance + clarity + usefulness) / 4;

    return {
      score: finalScore,
      metadata: {
        completeness,
        relevance,
        clarity,
        usefulness,
        reasoning,
      },
    };
  },
});
```

### Custom Scorer 3: Tool Call Efficiency

```typescript
// tests/evals/scorers/efficiency.ts
import { createScorer } from 'evalite';

export const toolCallEfficiencyScorer = createScorer({
  name: 'Tool Call Efficiency',
  score: ({ output, metadata }) => {
    // Assumes metadata contains tool call information
    const toolCalls = metadata?.toolCalls || [];
    const expectedMinCalls = metadata?.expectedMinCalls || 1;
    const expectedMaxCalls = metadata?.expectedMaxCalls || 5;

    const actualCalls = toolCalls.length;

    // Perfect score if within expected range
    if (actualCalls >= expectedMinCalls && actualCalls <= expectedMaxCalls) {
      return {
        score: 1.0,
        metadata: {
          actualCalls,
          expectedRange: `${expectedMinCalls}-${expectedMaxCalls}`,
          toolsUsed: toolCalls.map((c: any) => c.name),
        },
      };
    }

    // Penalize if too few or too many calls
    const penalty = Math.abs(actualCalls - expectedMaxCalls) * 0.1;
    const score = Math.max(0, 1 - penalty);

    return {
      score,
      metadata: {
        actualCalls,
        expectedRange: `${expectedMinCalls}-${expectedMaxCalls}`,
        toolsUsed: toolCalls.map((c: any) => c.name),
        penalty,
      },
    };
  },
});
```

---

## Testing Checklist

### Phase 2: Tools Testing (Week 3-7)

**Week 3-5: Financial Tools**
- [ ] Unit test all 14 financial tools
- [ ] Test API client error handling
- [ ] Test schema validation for all tools
- [ ] Test rate limiting and retry logic
- [ ] Create tool accuracy evals for each tool
- [ ] Verify all tools return expected data structure

**Week 6-7: Search Tools**
- [ ] Unit test Google News RSS parser
- [ ] Unit test Anthropic Web Search integration
- [ ] Test hybrid search coordination
- [ ] Test search result formatting
- [ ] Create search quality eval

### Phase 3: Core Agent Testing (Week 8-11)

**Week 8-9: Agent Components**
- [ ] Unit test each specialized agent (Planning, Action, Validation, Answer)
- [ ] Test structured output parsing
- [ ] Test tool binding and execution
- [ ] Test error handling in each phase

**Week 10-11: Agent Orchestration**
- [ ] Integration test full agent workflow
- [ ] Test multi-step task execution
- [ ] Test loop termination conditions
- [ ] Test goal achievement detection
- [ ] Create end-to-end research evals

### Phase 4: CLI Testing (Week 12-14)

**Week 12-13: UI Components**
- [ ] Unit test UI class methods
- [ ] Test spinner lifecycle
- [ ] Test color output
- [ ] Test cross-platform compatibility

**Week 14: CLI Integration**
- [ ] Test command history persistence
- [ ] Test interactive prompt flow
- [ ] Test exit handling
- [ ] Integration test CLI → Agent → UI flow

### Phase 5: Evaluation Framework (Week 15-18)

**Week 15-16: Dataset & Evaluators**
- [ ] Test CSV dataset loading
- [ ] Test LangSmith integration
- [ ] Test custom evaluator execution
- [ ] Verify all 50 test cases load correctly

**Week 17-18: Full Evaluation Suite**
- [ ] Run financial correctness eval (50 cases)
- [ ] Run answer quality eval (LLM-as-judge)
- [ ] Run tool accuracy eval
- [ ] Run cross-model consistency eval
- [ ] Compare TypeScript vs Python scores
- [ ] Generate evaluation report

### Phase 6-8: Final Testing

**Phase 6: Integration Testing**
- [ ] Full system integration tests
- [ ] Performance benchmarking
- [ ] Error handling validation
- [ ] Edge case testing

**Phase 7: Optimization Testing**
- [ ] Performance regression tests
- [ ] Caching effectiveness tests
- [ ] Parallel execution tests

**Phase 8: Production Readiness**
- [ ] Security testing
- [ ] Load testing
- [ ] Monitoring validation
- [ ] Final evaluation suite run

---

## CI/CD Integration

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
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'pnpm'

      - run: pnpm install
      - run: pnpm test
      - run: pnpm test:coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  evals:
    runs-on: ubuntu-latest
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    steps:
      - uses: actions/checkout@v3
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'pnpm'

      - run: pnpm install
      - run: pnpm rebuild better-sqlite3
      - run: pnpm eval
      - run: pnpm eval:export

      - name: Upload eval results
        uses: actions/upload-artifact@v3
        with:
          name: eval-results
          path: eval-results/
```

---

## Best Practices

### 1. Test Organization
- Keep unit tests close to source files
- Group integration tests by workflow
- Organize evals by evaluation type (correctness, quality, efficiency)

### 2. Test Data Management
```typescript
// tests/fixtures/financial-data.ts
export const mockFinancialData = {
  incomeStatement: {
    ticker: 'AAPL',
    period: 'Q4 2023',
    revenue: 119_600_000_000,
    netIncome: 33_900_000_000,
  },
  // ... more mock data
};
```

### 3. Eval Data Versioning
```
tests/evals/data/
├── v1/
│   └── financial-research-50.csv
├── v2/
│   └── financial-research-100.csv
└── current -> v2/
```

### 4. Continuous Evaluation
```bash
# Run evals on every commit
git commit -m "feat: improve tool accuracy"
pnpm eval
pnpm eval:export

# Compare scores to baseline
# Fail CI if scores drop > 10%
```

### 5. LLM-as-Judge Best Practices
- Use consistent evaluation prompts
- Include clear scoring rubrics
- Request detailed reasoning
- Use structured outputs (Zod schemas)
- Version your judge prompts

---

## Success Metrics

### Unit Test Coverage
- **Target**: 80%+ code coverage
- **Critical paths**: 95%+ coverage
- **Tools**: 100% coverage (all tools tested)

### Integration Test Coverage
- **Agent workflows**: All 4 phases tested
- **Multi-tool scenarios**: 10+ test cases
- **Error paths**: All error handlers tested

### Evaluation Scores
- **Financial correctness**: 90%+ accuracy
- **Answer quality**: 0.85+ average score
- **Tool accuracy**: 95%+ for all tools
- **Cross-model consistency**: 80%+ agreement

### Performance Benchmarks
- **Tool execution**: < 500ms median
- **Full query**: < 30s median
- **Evaluation suite**: < 5 minutes total

---

**END OF TESTING TEMPLATES**
