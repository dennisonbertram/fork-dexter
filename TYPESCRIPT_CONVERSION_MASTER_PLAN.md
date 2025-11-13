# Dexter TypeScript Conversion - Master Plan

**Date:** November 13, 2025
**Version:** 1.0
**Status:** Planning Complete - Ready for Implementation

---

## Executive Summary

This document serves as the comprehensive master plan for converting Dexter, an autonomous financial research agent, from Python to TypeScript. The conversion leverages **Vercel AI SDK** for multi-provider LLM integration and **Mastra AI Framework** for agent orchestration and workflow management.

**Total Components:** 5 major systems, 29 Python files, ~2,000 lines of code
**Estimated Timeline:** 20-24 weeks (5-6 months)
**Recommended Approach:** Phased migration with continuous testing

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Foundation Libraries](#foundation-libraries)
3. [Component Conversion Plans](#component-conversion-plans)
   - [Core Agent System](#1-core-agent-system)
   - [Financial Tools](#2-financial-tools)
   - [CLI & UI](#3-cli--ui)
   - [Evaluation Framework](#4-evaluation-framework)
   - [Search Tools](#5-search-tools)
4. [Implementation Roadmap](#implementation-roadmap)
5. [Risk Assessment](#risk-assessment)
6. [Success Metrics](#success-metrics)

---

## Architecture Overview

### Current Python Architecture

```
Python (LangChain)
├── Agent (Manual Loop)
│   ├── Planning Agent
│   ├── Action Agent
│   ├── Validation Agent
│   └── Answer Agent
├── Tools (14 financial + 1 search)
├── CLI (prompt_toolkit)
├── Evaluation (LangSmith Python SDK)
└── Model (OpenAI API)
```

### Target TypeScript Architecture

```
TypeScript (AI SDK + Mastra)
├── Core Agent (Mastra Agent + Workflows)
│   ├── Planning Agent (Structured Output)
│   ├── Action Agent (Tool Loop)
│   ├── Validation Agent (Structured Output)
│   └── Answer Agent (Structured Output)
├── Tools (AI SDK tool() pattern)
│   ├── Financial Tools (14 tools)
│   └── Search Tools (2 tools)
├── CLI (@inquirer/prompts + ora + chalk + custom history)
├── Evaluation (LangSmith TS SDK + Custom Evaluators)
└── Model (AI SDK multi-provider)
```

### Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| **Vercel AI SDK** over Anthropic SDK | Multi-provider support, larger ecosystem (2,300+ examples), unified interface |
| **Mastra + AI SDK hybrid** | Best orchestration patterns + full control when needed |
| **LangSmith** for evaluation | Mature dataset management, persistent tracking, production-ready |
| **@inquirer/prompts** for CLI input | Modern ESM-first, type-safe, 2M+ weekly downloads |
| **ora** for spinners | Industry standard, async-friendly, real-time updates |
| **chalk** for colors | 50M+ weekly downloads, excellent TypeScript support |
| **Hybrid search** (Google News + Web Search) | Complementary strengths - dated sources + real-time data |

---

## Foundation Libraries

### Required Dependencies

```json
{
  "dependencies": {
    "ai": "^7.0.0",
    "@ai-sdk/anthropic": "^0.0.50",
    "@ai-sdk/openai": "^0.0.50",
    "@mastra/core": "^0.10.0",
    "zod": "^3.22.0",
    "@inquirer/prompts": "^7.0.0",
    "chalk": "^5.3.0",
    "ora": "^8.0.0",
    "langsmith": "^0.3.0",
    "csv-parse": "^5.5.0",
    "rss-parser": "^3.13.0",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/node": "^20.11.0",
    "tsx": "^4.19.0",
    "vitest": "^2.1.0",
    "evalite": "^0.19.0",
    "autoevals": "^0.0.77",
    "@vitest/ui": "^2.1.0",
    "eslint": "^8.57.0",
    "prettier": "^3.2.0"
  }
}
```

### Environment Variables

```env
# LLM Providers
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...

# Financial Data
FINANCIAL_DATASETS_API_KEY=...

# Evaluation & Monitoring
LANGSMITH_API_KEY=ls_...
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_PROJECT=dexter-typescript
LANGSMITH_TRACING=true
```

---

## Component Conversion Plans

## 1. Core Agent System

**Document:** `FOUNDATION_ANALYSIS.md` (Section: Core Agent Conversion)
**Lines of Code:** ~280 (Python) → ~400 (TypeScript, with types)
**Complexity:** High
**Timeline:** 4 weeks

### Components

1. **Agent.py** → **core-agent.ts**
   - Main orchestration class
   - 9 methods: plan_tasks, ask_for_actions, ask_if_done, is_goal_achieved, optimize_tool_args, run, _generate_answer, _execute_tool, confirm_action
   - Manual execution loop with safety limits

2. **Model.py** → **model.ts**
   - LLM calling interface with retry logic
   - Structured output support
   - Tool binding

3. **Schemas.py** → **schemas.ts**
   - 5 Pydantic models → 5 Zod schemas
   - Task, TaskList, IsDone, Answer, OptimizedToolArgs

4. **Prompts.py** → **prompts.ts**
   - 8 system prompts with date injection helpers

### Recommended Approach: Mastra Agent + Custom Workflow Steps

**Why Hybrid?**
- Mastra provides agent infrastructure (tool calling, context, memory)
- Custom orchestration maintains Python's 4-phase pattern
- Better control over validation and optimization steps

**Implementation Pattern:**

```typescript
export class DexterAgent {
  private planningAgent: Agent;
  private actionAgent: Agent;
  private validationAgent: Agent;
  private answerAgent: Agent;

  async run(query: string): Promise<string> {
    // Phase 1: Task Planning
    const tasks = await this.planTasks(query);

    // Phase 2: Task Execution Loop
    for (const task of tasks) {
      while (!task.done) {
        const toolCalls = await this.askForActions(task);
        await this.executeToolCalls(toolCalls);
        task.done = await this.askIfDone(task);
      }
      if (await this.isGoalAchieved(query)) break;
    }

    // Phase 3: Answer Generation
    return await this.generateAnswer(query);
  }
}
```

**Safety Mechanisms:**
- Global step limit: 20
- Per-task step limit: 5
- Repetitive action detection (4-action sliding window)
- Argument optimization with GPT-4.1

**Key Files:**
- `/src/agent/core-agent.ts` - Main agent class
- `/src/agent/model.ts` - LLM interface
- `/src/agent/schemas.ts` - Zod schemas
- `/src/agent/prompts.ts` - System prompts

---

## 2. Financial Tools

**Document:** Financial Tools Conversion Plan
**Lines of Code:** ~600 (Python) → ~800 (TypeScript)
**Complexity:** Medium
**Timeline:** 3 weeks

### Tools Inventory (14 Total)

**Financial Statements (3):**
- `getIncomeStatements` - Revenue, expenses, net income
- `getBalanceSheets` - Assets, liabilities, equity
- `getCashFlowStatements` - Operating, investing, financing

**SEC Filings (4):**
- `getFilings` - Filing metadata
- `get10KFilingItems` - Annual report sections
- `get10QFilingItems` - Quarterly report sections
- `get8KFilingItems` - Current report sections

**Market Data (2):**
- `getPriceSnapshot` - Current stock price
- `getPrices` - Historical price data

**Metrics (2):**
- `getFinancialMetricsSnapshot` - Current metrics
- `getFinancialMetrics` - Historical metrics

**Analysis (2):**
- `getAnalystEstimates` - Analyst forecasts
- `getSegmentedRevenues` - Revenue by segment

**News (1):**
- `getNews` - Company news

### Implementation Pattern

**Standard Tool Template:**

```typescript
import { tool } from 'ai';
import { z } from 'zod';

export const getStockPrice = tool({
  description: 'Get current stock price for a ticker',
  inputSchema: z.object({
    ticker: z.string().describe('Stock ticker symbol'),
  }),
  execute: async ({ ticker }) => {
    const response = await apiClient.get(`/prices/${ticker}`);
    return { ticker, price: response.price };
  },
});
```

**API Client Architecture:**

```typescript
export class FinancialDatasetsClient {
  private baseUrl = 'https://api.financialdatasets.ai';
  private apiKey: string;

  async get<T>(endpoint: string, params?: Record<string, any>): Promise<T> {
    const url = new URL(endpoint, this.baseUrl);
    // Add params, handle errors, retry logic
    const response = await fetch(url, {
      headers: { 'x-api-key': this.apiKey }
    });
    return response.json();
  }
}
```

**Key Files:**
- `/src/tools/finance/api.ts` - Centralized API client
- `/src/tools/finance/fundamentals.ts` - Financial statement tools
- `/src/tools/finance/filings.ts` - SEC filing tools
- `/src/tools/finance/metrics.ts` - Metrics tools
- `/src/tools/finance/prices.ts` - Price data tools
- `/src/tools/finance/constants.ts` - SEC filing mappings
- `/src/tools/index.ts` - Tool registry

**Migration Checklist:** 8 phases, 5 weeks

---

## 3. CLI & UI

**Document:** CLI & UI Conversion Plan
**Lines of Code:** ~400 (Python) → ~500 (TypeScript)
**Complexity:** Medium
**Timeline:** 3-4 weeks

### Components

1. **CLI (cli.py)** → **cli/index.ts**
   - Interactive prompt with history
   - Command persistence (`.dexter_history`)
   - Exit handling (Ctrl+C, "exit", "quit")

2. **UI (ui.py)** → **ui/index.ts**
   - Colors class (ANSI codes)
   - Spinner class (animated loading)
   - UI class (10 display methods)
   - Progress decorator pattern

3. **Logger (logger.py)** → **utils/logger.ts**
   - Logging facade wrapping UI
   - Message history
   - Progress context manager

4. **Intro (intro.py)** → **utils/intro.ts**
   - ASCII art welcome screen

### Library Selection

**See: CLI_FRAMEWORK_DECISION.md for complete framework standardization**

| Python | TypeScript | Rationale |
|--------|-----------|-----------|
| `prompt_toolkit.PromptSession` | `@inquirer/prompts` v7 | Modern ESM-first, type-safe, 2M+ weekly downloads |
| `prompt_toolkit.FileHistory` | Custom `CommandHistory` class | File-based persistence (~/.dexter_history) |
| Manual ANSI codes | `chalk` v5 | Industry standard, 50M+ weekly downloads |
| Threading (Spinner) | `ora` v8 | Async-friendly, event-loop based animations |
| Manual decorators | TypeScript decorators | Native language feature |

### Implementation Highlights

**Interactive Prompt with History:**

```typescript
import { input, confirm } from '@inquirer/prompts';
import { CommandHistory } from './history.js';

export class InteractivePrompt {
  private history: CommandHistory;

  constructor() {
    this.history = new CommandHistory('~/.dexter_history');
  }

  async prompt(): Promise<string> {
    const answer = await input({
      message: 'Ask Dexter a question:'
    });
    await this.history.add(answer.trim());
    return answer.trim();
  }

  async confirmContinue(): Promise<boolean> {
    return await confirm({
      message: 'Continue with another query?',
      default: true
    });
  }
}
```

**Animated Spinner:**

```typescript
export class Spinner {
  private oraSpinner: Ora;

  constructor(message: string, color: string) {
    this.oraSpinner = ora({ text: message, color });
  }

  updateMessage(message: string): void {
    this.oraSpinner.text = message;
  }

  stop(finalMessage: string, symbol: string): void {
    this.oraSpinner.stopAndPersist({ symbol, text: finalMessage });
  }
}
```

**Progress Decorator:**

```typescript
export function showProgress(message: string, successMessage: string) {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const spinner = new Spinner(message);
      spinner.start();

      try {
        const result = await originalMethod.apply(this, args);
        spinner.stop(successMessage, '✓');
        return result;
      } catch (error) {
        spinner.stop(`Failed: ${error.message}`, '✗');
        throw error;
      }
    };

    return descriptor;
  };
}
```

**Key Files:**
- `/src/cli/index.ts` - Main CLI entry point
- `/src/cli/prompt.ts` - Interactive prompt
- `/src/cli/history.ts` - Command history persistence
- `/src/ui/spinner.ts` - Animated spinner
- `/src/ui/colors.ts` - Color definitions
- `/src/ui/index.ts` - UI class with display methods
- `/src/utils/logger.ts` - Logging facade

**Migration Timeline:** 3-4 weeks across 6 phases

---

## 4. Evaluation Framework

**Document:** Evaluation Framework Conversion Plan
**Lines of Code:** ~300 (Python) → ~350 (TypeScript)
**Complexity:** Medium
**Timeline:** 4 weeks

### Components

1. **Evaluator (evaluator.py)** → **evaluator.ts**
   - `evalCorrectness()` - Scores agent answer vs reference
   - Structured output with Zod + AI SDK
   - GPT-4.1 for evaluation scoring

2. **Dataset (dataset.py)** → **dataset.ts**
   - `createDatasetFromCsv()` - Loads CSV to LangSmith
   - Example transformation

3. **Data Loader (loader.py)** → **data/loader.ts**
   - CSV parsing with `csv-parse`
   - Type-safe data loading

4. **Prompts (prompts.py)** → **prompts.ts**
   - `CORRECTNESS_PROMPT` with scoring guidelines

5. **Run Evaluate (run_evaluate.py)** → **run-evaluate.ts**
   - Main execution script
   - 50-question evaluation dataset

### Recommended Approach: Hybrid (LangSmith + Custom Evaluators)

**Why Hybrid?**
- **LangSmith** for dataset management, experiment tracking, persistent storage
- **Custom TypeScript evaluators** for domain-specific financial evaluation
- Direct migration path from Python (API parity)

**Implementation Pattern:**

```typescript
// Evaluator with structured output
export async function evalCorrectness(
  run: { inputs: EvalInputs; outputs: EvalOutputs },
  example: { outputs: EvalOutputs }
): Promise<{ key: string; score: number; comment: string }> {
  const { object: evalResult } = await generateObject({
    model: openai('gpt-4.1'),
    schema: CorrectnessScoreSchema,
    messages: [{ role: 'user', content: CORRECTNESS_PROMPT }]
  });

  return {
    key: 'correctness_score',
    score: evalResult.score / 5, // Normalize to 0-1
    comment: evalResult.reasoning
  };
}

// Dataset creation from CSV
export async function createDatasetFromCsv(
  csvPath: string,
  datasetName: string
): Promise<string> {
  const loader = new DataLoader(csvPath);
  const data = await loader.load();
  const client = new Client();

  const dataset = await client.createDataset(datasetName);

  await client.createExamples({
    inputs: data.map(row => ({ question: row.Question })),
    outputs: data.map(row => ({ answer: row.Answer })),
    metadata: data.map(row => ({
      questionType: row['Question Type'],
      expertTimeMins: row['Expert time (mins)'],
      rubric: row.Rubric
    })),
    datasetId: dataset.id
  });

  return datasetName;
}

// Evaluation runner
export async function runEvaluation({
  datasetName,
  experimentPrefix = 'Dexter Finance Agent Eval',
  maxConcurrency = 5
}: RunEvaluationOptions) {
  const target = createTargetFunction(); // Wraps agent.run()

  const results = await evaluate(target, {
    data: datasetName,
    evaluators: [evalCorrectness],
    experimentPrefix,
    maxConcurrency
  });

  return results;
}
```

**Key Files:**
- `/src/evals/dataset.ts` - Dataset creation from CSV
- `/src/evals/evaluator.ts` - Correctness evaluator with structured output
- `/src/evals/runner.ts` - Evaluation orchestration
- `/src/evals/prompts.ts` - Evaluation prompts
- `/src/evals/data/loader.ts` - CSV parser
- `/src/evals/data/vals-finance-agent-50.csv` - 50 test cases

**Migration Timeline:** 6 phases, 6 weeks

---

## 5. Search Tools

**Document:** Search Tools Conversion Plan
**Lines of Code:** ~200 (Python) → ~300 (TypeScript)
**Complexity:** Medium
**Timeline:** 3-4 weeks

### Recommended Approach: Hybrid (Google News + Anthropic Web Search)

**Why Both?**

| Feature | Google News RSS | Anthropic Web Search |
|---------|----------------|---------------------|
| **Publication Dates** | ✅ Reliable | ⚠️ Limited |
| **Coverage** | News Only | Entire Web |
| **API Key Required** | ❌ No | ✅ Yes |
| **Cost** | Free | API Tokens |
| **Real-time Data** | News Only | ✅ Yes |
| **Domain Filtering** | ❌ No | ✅ Yes |

**Complementary Use Cases:**
- **Google News** for: Recent news, earnings reports, company announcements
- **Web Search** for: Current prices, real-time market data, general research

### Implementation Pattern

**Google News Tool:**

```typescript
import Parser from 'rss-parser';

export const searchGoogleNews = tool({
  description: 'Search Google News for financial news articles',
  inputSchema: z.object({
    query: z.string(),
    maxResults: z.number().default(5)
  }),
  execute: async ({ query, maxResults }) => {
    const parser = new Parser();
    const feed = await parser.parseURL(
      `https://news.google.com/rss/search?q=${encodeURIComponent(query)}`
    );

    const items = feed.items.slice(0, maxResults);

    // Resolve redirect URLs in parallel
    const resolvedItems = await Promise.all(
      items.map(async item => ({
        title: item.title,
        url: await resolveGoogleNewsUrl(item.link),
        publishedDate: new Date(item.pubDate)
      }))
    );

    return { results: resolvedItems };
  }
});
```

**Anthropic Web Search Tool:**

```typescript
import { anthropic } from '@ai-sdk/anthropic';
import { generateText } from 'ai';

export const webSearchTool = anthropic.tools.webSearch_20250305({
  maxUses: 5,
  allowedDomains: ['sec.gov', 'finance.yahoo.com', 'bloomberg.com'],
  userLocation: {
    type: 'approximate',
    country: 'US'
  }
});

// Use in agent
const result = await generateText({
  model: anthropic('claude-opus-4-20250514'),
  tools: { web_search: webSearchTool },
  prompt: 'Find latest Apple financial news'
});
```

**Key Files:**
- `/src/tools/search/google-news.ts` - RSS parsing with URL resolution
- `/src/tools/search/web-search.ts` - Anthropic Web Search wrapper
- `/src/tools/search/utils.ts` - Shared utilities
- `/src/tools/search/types.ts` - SearchResult interface

**Migration Timeline:** 4 phases, 3-4 weeks

---

## Implementation Roadmap

### Phase Overview

| Phase | Duration | Components | Milestone |
|-------|----------|-----------|-----------|
| **Phase 1: Foundation** | 2 weeks | Project setup, dependencies, core schemas | TypeScript project initialized |
| **Phase 2: Tools** | 5 weeks | All 15 tools (14 financial + 1 search) | Tools functional and tested |
| **Phase 3: Core Agent** | 4 weeks | Agent orchestration, model interface | Agent runs queries end-to-end |
| **Phase 4: CLI & UI** | 3 weeks | Interactive prompt, spinners, logging | Interactive CLI working |
| **Phase 5: Evaluation** | 4 weeks | Dataset management, evaluators | Evaluation suite running |
| **Phase 6: Integration** | 2 weeks | Full system integration testing | System feature-complete |
| **Phase 7: Optimization** | 2 weeks | Performance tuning, caching | Production-ready |
| **Phase 8: Deployment** | 2 weeks | CI/CD, monitoring, documentation | Deployed to production |

**Total Timeline:** 24 weeks (6 months)

### Detailed Phase Breakdown

#### Phase 1: Foundation (Weeks 1-2)

**Week 1:**
- [ ] Initialize TypeScript project with proper tsconfig
- [ ] Install all dependencies (see CLI_FRAMEWORK_DECISION.md for standardized CLI framework)
  - Core: `ai`, `@ai-sdk/anthropic`, `@ai-sdk/openai`, `@mastra/core`, `zod`
  - CLI: `@inquirer/prompts`, `chalk`, `ora`
  - Data: `csv-parse`, `rss-parser`
  - Evaluation: `langsmith`
- [ ] Set up development environment (.env, scripts)
- [ ] Create directory structure
- [ ] Configure ESLint, Prettier, Vitest
- [ ] Set up Git repository and branching strategy

**Week 2:**
- [ ] Convert all Pydantic schemas to Zod schemas
- [ ] Port system prompts to TypeScript
- [ ] Create shared type definitions
- [ ] Set up LangSmith project
- [ ] Configure API keys and environment variables
- [ ] Write initial documentation (README, CONTRIBUTING)
- [ ] Review CLI_FRAMEWORK_DECISION.md for CLI implementation patterns

**Deliverables:**
- Working TypeScript project skeleton
- All schemas and prompts converted
- Development environment configured

---

#### Phase 2: Tools (Weeks 3-7)

**Reference: TESTING_TEMPLATES.md for unit test patterns and tool accuracy evals**

**Week 3: API Client + Financial Statements**
- [ ] Implement `FinancialDatasetsClient` class
- [ ] Create API error types and handling
- [ ] Convert `getIncomeStatements` tool
- [ ] Convert `getBalanceSheets` tool
- [ ] Convert `getCashFlowStatements` tool
- [ ] Write unit tests for API client (see Template 1: Tool Function Test)
- [ ] Write unit tests for statement tools (see Template 1: Tool Function Test)

**Week 4: SEC Filings**
- [ ] Convert SEC filing constants (10-K, 10-Q, 8-K mappings)
- [ ] Convert `getFilings` tool
- [ ] Convert `get10KFilingItems` tool
- [ ] Convert `get10QFilingItems` tool
- [ ] Convert `get8KFilingItems` tool
- [ ] Write unit tests for filing tools

**Week 5: Market Data & Metrics**
- [ ] Convert `getPriceSnapshot` tool
- [ ] Convert `getPrices` tool
- [ ] Convert `getFinancialMetricsSnapshot` tool
- [ ] Convert `getFinancialMetrics` tool
- [ ] Write unit tests for price/metrics tools

**Week 6: Analysis & News**
- [ ] Convert `getAnalystEstimates` tool
- [ ] Convert `getSegmentedRevenues` tool
- [ ] Convert `getNews` tool
- [ ] Write unit tests for analysis tools

**Week 7: Search Tools**
- [ ] Implement Google News RSS parser
- [ ] Implement URL redirect resolution
- [ ] Integrate Anthropic Web Search tool
- [ ] Create tool registry
- [ ] Write integration tests for all tools
- [ ] Test API rate limiting and error handling

**Deliverables:**
- All 15 tools converted and tested
- Comprehensive test coverage (>80%)
- Tool registry for agent integration
- Tool accuracy evals created (see Template 8: Tool Accuracy Eval)

---

#### Phase 3: Core Agent (Weeks 8-11)

**Week 8: Model Interface & Schemas**
- [ ] Implement `callLLM()` function with retry logic
- [ ] Add structured output support (Zod integration)
- [ ] Add tool binding support
- [ ] Test LLM calling with different providers
- [ ] Implement error handling and timeouts

**Week 9: Agent Foundation**
- [ ] Create `DexterAgent` class structure
- [ ] Initialize specialized agents (Planning, Action, Validation, Answer)
- [ ] Implement state management (AgentState interface)
- [ ] Add safety mechanisms (step limits, loop detection)

**Week 10: Agent Methods**
- [ ] Implement `planTasks()` method
- [ ] Implement `askForActions()` method
- [ ] Implement `executeToolCall()` method
- [ ] Implement `optimizeToolArgs()` method
- [ ] Implement `askIfDone()` method
- [ ] Implement `isGoalAchieved()` method
- [ ] Implement `generateAnswer()` method

**Week 11: Agent Orchestration**
- [ ] Implement main `run()` loop
- [ ] Implement `executeTask()` loop
- [ ] Add comprehensive error handling
- [ ] Add progress logging
- [ ] Write unit tests for each method (see Template 2: Schema Validation Test)
- [ ] Write integration tests for full agent flow (see Template 4: Agent Workflow Test)

**Deliverables:**
- Fully functional agent that can execute queries end-to-end
- Comprehensive test coverage (see Template 4 & 5: Integration Tests)
- Performance benchmarks vs Python version

---

#### Phase 4: CLI & UI (Weeks 12-14)

**Reference: CLI_FRAMEWORK_DECISION.md for complete implementation patterns**
**Reference: TESTING_TEMPLATES.md Template 3 for UI testing**

**Week 12: UI Components**
- [ ] Implement `Colors` class (chalk v5 wrapper)
- [ ] Implement `Spinner` class (ora v8 wrapper)
- [ ] Implement `UI` class (all 10 display methods)
- [ ] Implement box-drawing and formatting
- [ ] Test cross-platform compatibility (Mac, Linux, Windows)
- [ ] Write unit tests for UI components (see Template 3: Utility Function Test)

**Week 13: CLI Implementation**
- [ ] Implement `CommandHistory` class with file persistence (~/.dexter_history)
- [ ] Implement `InteractivePrompt` using @inquirer/prompts v7
- [ ] Create main CLI loop with input() and confirm() prompts
- [ ] Add exit handling (Ctrl+C, "exit", "quit")
- [ ] Implement welcome screen (intro.ts)
- [ ] Test CLI interaction flow

**Week 14: Logger & Integration**
- [ ] Implement `Logger` class (UI facade)
- [ ] Add message history
- [ ] Implement progress decorators
- [ ] Integrate CLI with agent
- [ ] Add streaming UI updates during agent execution
- [ ] Test end-to-end CLI → Agent → UI flow
- [ ] Verify all patterns match CLI_FRAMEWORK_DECISION.md

**Deliverables:**
- Fully interactive CLI with rich UI
- Command history persistence
- Seamless integration with agent system

---

#### Phase 5: Evaluation (Weeks 15-18)

**Reference: TESTING_TEMPLATES.md for Evalite setup and evaluation patterns**

**Week 15: Evalite Setup & Dataset Management**
- [ ] Install Evalite, Vitest, Autoevals (see TESTING_TEMPLATES.md § Evalite Setup)
- [ ] Configure evalite.config.ts
- [ ] Implement `DataLoader` class (CSV parsing)
- [ ] Set up LangSmith dataset
- [ ] Load 50-question evaluation dataset
- [ ] Verify dataset in LangSmith UI

**Week 16: Custom Scorers & Evaluators**
- [ ] Implement financial correctness scorer (see Custom Scorer 1)
- [ ] Implement answer quality scorer (see Custom Scorer 2)
- [ ] Implement tool call efficiency scorer (see Custom Scorer 3)
- [ ] Test scorers with sample questions
- [ ] Add error handling for evaluation failures

**Week 17: Evaluation Tests**
- [ ] Create financial research eval (see Template 6)
- [ ] Create answer quality eval (see Template 7)
- [ ] Create tool accuracy eval (see Template 8)
- [ ] Create cross-model consistency eval (see Template 9)
- [ ] Test evaluation on 3 examples with `pnpm eval:dev`

**Week 18: Full Evaluation Suite**
- [ ] Run evaluation on all 50 questions with `pnpm eval`
- [ ] Analyze results in Evalite UI (localhost:3006)
- [ ] Compare TypeScript vs Python scores
- [ ] Export results as static HTML with `pnpm eval:export`
- [ ] Document evaluation process and findings

**Deliverables:**
- Complete Evalite evaluation framework
- Custom financial scorers (correctness, quality, efficiency)
- Results comparison (TypeScript vs Python)
- Quality metrics validation (>90% correctness, >0.85 quality)
- Static HTML evaluation report for CI/CD

---

#### Phase 6: Integration (Weeks 19-20)

**Week 19: System Integration**
- [ ] End-to-end testing with real queries
- [ ] Performance profiling and bottleneck identification
- [ ] Memory leak detection and fixes
- [ ] Cross-component integration testing
- [ ] Error propagation and recovery testing

**Week 20: Refinement**
- [ ] Fix integration bugs
- [ ] Optimize hot paths
- [ ] Add comprehensive logging
- [ ] Improve error messages
- [ ] Update documentation

**Deliverables:**
- Fully integrated system
- All components working together seamlessly
- Bug-free operation

---

#### Phase 7: Optimization (Weeks 21-22)

**Week 21: Performance Tuning**
- [ ] Implement caching for tool results
- [ ] Add LRU cache for LLM responses
- [ ] Optimize parallel tool execution
- [ ] Profile and reduce memory usage
- [ ] Benchmark against Python version

**Week 22: Advanced Features**
- [ ] Add streaming response support
- [ ] Implement progress context managers
- [ ] Add telemetry and monitoring
- [ ] Create performance dashboards
- [ ] Document optimization techniques

**Deliverables:**
- 20-30% performance improvement over Python
- Reduced token usage
- Better user experience with streaming

---

#### Phase 8: Deployment (Weeks 23-24)

**Week 23: CI/CD Setup**
- [ ] Set up GitHub Actions workflows
- [ ] Configure automated testing
- [ ] Add code coverage reporting
- [ ] Set up Docker containerization
- [ ] Create deployment scripts

**Week 24: Production Deployment**
- [ ] Deploy to staging environment
- [ ] Run final evaluation suite
- [ ] Deploy to production
- [ ] Set up monitoring and alerting
- [ ] Create runbook and troubleshooting guides
- [ ] Train team on TypeScript version

**Deliverables:**
- Production-ready deployment
- Automated CI/CD pipeline
- Comprehensive documentation

---

## Risk Assessment

### High Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| **Agent Quality Degradation** | Medium | High | Run evaluation suite continuously; maintain parity benchmarks |
| **API Rate Limiting** | Medium | Medium | Implement exponential backoff; add caching layer |
| **Tool Execution Failures** | Medium | Medium | Comprehensive error handling; fallback mechanisms |
| **Streaming Implementation** | Medium | Medium | Start with batch mode; add streaming incrementally |

### Medium Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| **TypeScript Learning Curve** | Low | Medium | Provide training materials; pair programming sessions |
| **Library Breaking Changes** | Medium | Low | Lock dependency versions; comprehensive testing |
| **Performance Regression** | Low | Medium | Continuous benchmarking; profiling tools |

### Low Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| **CLI Compatibility Issues** | Low | Low | Test on multiple platforms; use standard libraries |
| **Evaluation Drift** | Low | Low | Use same dataset; validate metrics |

---

## Success Metrics

### Functional Metrics

- [ ] **Feature Parity**: 100% of Python features replicated
- [ ] **Tool Success Rate**: ≥95% successful tool executions
- [ ] **Agent Accuracy**: Evaluation score within 5% of Python version
- [ ] **Error Recovery**: Graceful handling of 100% of error scenarios

### Performance Metrics

- [ ] **Response Time**: ≤120% of Python version (ideally better)
- [ ] **Token Usage**: ≤110% of Python version
- [ ] **Memory Usage**: ≤150% of Python version
- [ ] **Startup Time**: <2 seconds (cold start)

### Quality Metrics

- [ ] **Test Coverage**: ≥80% code coverage
- [ ] **Type Safety**: 0 TypeScript errors with strict mode
- [ ] **Documentation**: 100% of public APIs documented
- [ ] **Lint/Format**: 0 ESLint errors, 100% Prettier formatted

### User Experience Metrics

- [ ] **CLI Responsiveness**: Real-time progress updates
- [ ] **Error Messages**: Clear, actionable error messages
- [ ] **Streaming**: Real-time token streaming (Phase 7)
- [ ] **History**: Command history working across sessions

---

## Technical Debt Management

### Known Trade-offs

1. **Streaming Support**: Deferred to Phase 7 (optimization)
   - Reason: Core functionality first, streaming is enhancement
   - Impact: Users wait for full response initially

2. **Multi-turn Conversations**: Not in MVP
   - Reason: Python version doesn't have it
   - Impact: Each query is independent

3. **Advanced Caching**: Added in Phase 7
   - Reason: Optimize after correctness validated
   - Impact: Higher token usage initially

### Future Enhancements

- [ ] Multi-turn conversation support
- [ ] Agent memory across sessions
- [ ] Custom tool creation via UI
- [ ] Web-based dashboard
- [ ] Multi-agent collaboration
- [ ] Fine-tuned models for financial domain

---

## Documentation Requirements

### Developer Documentation

- [ ] Architecture overview (this document)
- [ ] API documentation (JSDoc + TypeDoc)
- [ ] Tool creation guide
- [ ] Agent customization guide
- [ ] Deployment guide

### User Documentation

- [ ] User guide (CLI usage)
- [ ] Query examples and best practices
- [ ] Troubleshooting guide
- [ ] FAQ

### Operational Documentation

- [ ] Runbook (incident response)
- [ ] Monitoring and alerting setup
- [ ] Backup and recovery procedures
- [ ] Scaling guide

---

## Appendices

### A. File Structure

```
dexter-typescript/
├── src/
│   ├── agent/
│   │   ├── core-agent.ts
│   │   ├── model.ts
│   │   ├── schemas.ts
│   │   └── prompts.ts
│   ├── tools/
│   │   ├── finance/
│   │   │   ├── api.ts
│   │   │   ├── constants.ts
│   │   │   ├── fundamentals.ts
│   │   │   ├── filings.ts
│   │   │   ├── metrics.ts
│   │   │   ├── prices.ts
│   │   │   ├── news.ts
│   │   │   ├── estimates.ts
│   │   │   └── segments.ts
│   │   ├── search/
│   │   │   ├── google-news.ts
│   │   │   ├── web-search.ts
│   │   │   └── utils.ts
│   │   └── index.ts
│   ├── cli/
│   │   ├── index.ts
│   │   ├── prompt.ts
│   │   └── history.ts
│   ├── ui/
│   │   ├── index.ts
│   │   ├── spinner.ts
│   │   ├── colors.ts
│   │   └── decorators.ts
│   ├── evals/
│   │   ├── dataset.ts
│   │   ├── evaluator.ts
│   │   ├── runner.ts
│   │   ├── prompts.ts
│   │   └── data/
│   │       ├── loader.ts
│   │       └── vals-finance-agent-50.csv
│   ├── utils/
│   │   ├── logger.ts
│   │   └── intro.ts
│   └── types.ts
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/
│   ├── architecture.md
│   ├── api.md
│   └── user-guide.md
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── .env.example
└── README.md
```

### B. Migration Checklist

#### Pre-Migration
- [ ] Backup Python codebase
- [ ] Document current Python behavior
- [ ] Run Python evaluation suite and save baselines
- [ ] Set up TypeScript project repository

#### During Migration
- [ ] Convert one component at a time
- [ ] Test each component independently
- [ ] Run integration tests after each phase
- [ ] Compare metrics to Python baseline

#### Post-Migration
- [ ] Run full evaluation suite
- [ ] Compare all metrics to Python
- [ ] Performance benchmark
- [ ] User acceptance testing
- [ ] Deploy to production

### C. Support and Escalation

**Development Team:**
- Lead Developer: [Name]
- Backend Engineers: [Names]
- DevOps Engineer: [Name]

**Escalation Path:**
1. Team Lead (response: 4 hours)
2. Engineering Manager (response: 8 hours)
3. CTO (response: 24 hours)

**Support Channels:**
- Slack: #dexter-typescript
- Email: dexter-team@company.com
- Issues: GitHub Issues

---

## Conclusion

This master plan provides a comprehensive roadmap for converting Dexter from Python to TypeScript. The 24-week phased approach ensures incremental progress with continuous validation against the Python baseline. The hybrid architecture using Vercel AI SDK and Mastra provides the best balance of flexibility, type safety, and modern AI tooling.

**Next Steps:**
1. Review and approve this plan
2. Allocate development resources
3. Set up project infrastructure
4. Begin Phase 1 (Foundation)

**Success Factors:**
- Continuous testing and validation
- Frequent comparison to Python baseline
- Incremental delivery with working milestones
- Comprehensive documentation throughout

---

**Document Control**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-13 | AI Agent | Initial comprehensive plan |

**Related Documents:**
- `FOUNDATION_ANALYSIS.md` - Core agent conversion details
- `Financial Tools Conversion Plan` - Complete tool conversion guide
- `CLI & UI Conversion Plan` - Interactive interface details
- `Evaluation Framework Conversion Plan` - Testing and validation
- `Search Tools Conversion Plan` - Search functionality migration
