# TypeScript Conversion Foundation Analysis

## Overview

Converting Dexter from Python to TypeScript will leverage two primary foundational libraries:

1. **Vercel AI SDK** - Unified LLM abstraction layer with multi-provider support
2. **Mastra AI Framework** - Agent/workflow orchestration layer

---

## 1. Vercel AI SDK

### Purpose
Provider-agnostic TypeScript SDK for building AI-powered applications. Provides a unified interface to multiple LLM providers (Anthropic, OpenAI, Google, etc.) with built-in support for tool use, streaming, structured outputs, and agentic patterns.

### Key Advantages
- **Multi-provider**: Works with Claude, GPT-4, Gemini, and 50+ other models
- **Unified interface**: Same API regardless of provider
- **Rich ecosystem**: 2,300+ code snippets, extensive documentation
- **Built-in agents**: `ToolLoopAgent` and `createAgentUIStream` for autonomous tool execution
- **Streaming**: Native streaming support with progress events
- **Type safety**: Full TypeScript support with Zod integration
- **Easy provider switching**: Change providers by modifying one line

### Key Capabilities

#### Tool Definition
```typescript
import { tool } from 'ai';
import { z } from 'zod';

const getStockPrice = tool({
  description: 'Get current stock price for a ticker',
  inputSchema: z.object({
    ticker: z.string().describe('Stock ticker symbol'),
  }),
  execute: async ({ ticker }) => {
    const price = await fetchPrice(ticker);
    return { ticker, price };
  },
});
```

**Mapping to Dexter**: Replaces LangChain's `@tool` decorator pattern. Identical to Mastra's tool definition (AI SDK is actually the foundation for Mastra).

#### Streaming Text Generation
```typescript
import { anthropic } from '@ai-sdk/anthropic'; // or any provider
import { streamText } from 'ai';

const result = streamText({
  model: anthropic('claude-opus-4-20250514'),
  prompt: 'Write a story about AI',
});

// Event-based streaming
for await (const textDelta of result.textStream) {
  console.log(textDelta);
}

const finalText = await result.text;
```

**Benefits**: Provider-agnostic - same code works with OpenAI, Google, etc.

#### Tool-Augmented Generation (Automatic Tool Loop)
```typescript
import { generateText, tool } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

const result = await generateText({
  model: anthropic('claude-opus-4-20250514'),
  tools: {
    getWeather: tool({
      description: 'Get weather for a location',
      inputSchema: z.object({
        location: z.string(),
      }),
      execute: async ({ location }) => ({
        location,
        temp: 72
      }),
    }),
  },
  prompt: 'What is the weather in SF?',
  stopWhen: stepCountIs(5), // Safety limit
});
```

**Mapping to Dexter**: The `generateText` with `stopWhen` replaces the manual loop in `agent.py`.

#### Streaming Tool Execution
```typescript
import { streamText } from 'ai';

const result = streamText({
  model: anthropic('claude-opus-4-20250514'),
  tools: {
    getWeather: weatherTool,
    getNews: newsTool,
  },
  prompt: 'Find weather and news for NYC',
  toolCallStreaming: true, // Stream tool calls as they occur
});

// Monitor streaming
for await (const chunk of result.fullStream) {
  if (chunk.type === 'tool-call') {
    console.log('Tool called:', chunk.toolName);
  } else if (chunk.type === 'text-delta') {
    console.log('Text:', chunk.textDelta);
  }
}
```

**Benefits**: Real-time visibility into tool execution for interactive experiences.

#### Built-in Agent Loop
```typescript
import { ToolLoopAgent } from 'ai';

const agent = new ToolLoopAgent({
  model: anthropic('claude-opus-4-20250514'),
  tools: {
    getStockPrice: stockTool,
    getAnalystEstimates: estimatesTool,
  },
  system: 'You are a financial analyst. Use tools to research stocks.',
});

// Agent automatically loops until task complete
const result = await agent.execute({
  input: 'Analyze Apple stock for me',
});

console.log(result.text);
```

**Mapping to Dexter**: Replaces the manual `run()` loop in `agent.py`. Built-in safety limits and loop detection.

#### Structured Output Streaming
```typescript
import { streamObject } from 'ai';

const result = streamObject({
  model: anthropic('claude-opus-4-20250514'),
  schema: z.object({
    tasks: z.array(z.object({
      id: z.string(),
      description: z.string(),
      completed: z.boolean(),
    })),
  }),
  prompt: 'Plan my day',
});

// Stream structured output as it's generated
for await (const chunk of result.partialObjectStream) {
  console.log(JSON.stringify(chunk, null, 2));
}
```

**Mapping to Dexter**: Replaces Pydantic's structured output validation.

#### Multi-Provider Example
```typescript
// Same code, just change the provider
import { openai } from '@ai-sdk/openai';
// import { anthropic } from '@ai-sdk/anthropic';
// import { google } from '@ai-sdk/google';

const result = await streamText({
  model: openai('gpt-4o'), // Switch providers here
  tools: { /* same tools */ },
  prompt: 'Your prompt',
});
```

**Benefit**: Easy A/B testing, cost optimization, or provider switching.

### Advanced Features

#### Web Fetch Tool (Anthropic)
```typescript
const result = await generateText({
  model: anthropic('claude-opus-4-20250514'),
  tools: {
    web_fetch: anthropic.tools.webFetch_20250910({ maxUses: 1 }),
  },
  prompt: 'What is this page about? https://example.com',
});
```

**Mapping to Dexter**: Could replace Google News search integration.

#### Web Search Tool (Anthropic)
```typescript
const webSearchTool = anthropic.tools.webSearch_20250305({
  maxUses: 5,
  allowedDomains: ['crunchbase.com', 'sec.gov'],
  blockedDomains: ['spam-site.com'],
});

const result = await generateText({
  model: anthropic('claude-opus-4-20250514'),
  tools: { web_search: webSearchTool },
  prompt: 'Find latest Apple financial news',
});
```

**Mapping to Dexter**: Enhanced news retrieval with built-in web search.

#### Code Execution Tool (Anthropic)
```typescript
const codeExecutionTool = anthropic.tools.codeExecution_20250825();

const result = await generateText({
  model: anthropic('claude-opus-4-20250514'),
  tools: { code_execution: codeExecutionTool },
  prompt: 'Calculate mean and std dev of [1,2,3,4,5]',
});
```

**Potential use**: Financial calculations on retrieved data.

#### Computer Use Tool (Anthropic)
```typescript
const computerTool = anthropic.tools.computer_20241022({
  displayWidthPx: 1920,
  displayHeightPx: 1080,
  execute: async ({ action, coordinate, text }) => {
    // Handle keyboard/mouse actions
    if (action === 'screenshot') {
      return { type: 'image', data: getScreenshot() };
    }
    return `executed ${action}`;
  },
});
```

**Future use**: Could automate financial website interaction.

### Provider Ecosystem

| Provider | Model | Use Case |
|----------|-------|----------|
| Anthropic | Claude 3.5 Sonnet | Primary reasoning and tool use |
| OpenAI | GPT-4o | Fallback, cost optimization |
| Google | Gemini 2.5 Flash | Cost-effective alternative |
| Mistral | Large, Medium | European alternative |
| Groq | LLaMA 3 | Fast inference |

**Strategy**: Use Claude for complex financial reasoning, OpenAI for cost-sensitive queries.

### Advantages Over Anthropic SDK Alone
- **Multi-provider**: Not locked into Claude
- **Larger ecosystem**: 2,300+ code examples, more community support
- **Built-in agents**: `ToolLoopAgent` handles loops automatically
- **Structured output**: Native support via `streamObject`
- **Provider-specific tools**: Easy access to Claude's web search, code execution, etc.
- **Abstracts differences**: Handles provider variations automatically

### Limitations
- **One abstraction layer**: Slightly less control than native SDK
- **Learning curve**: More concepts to learn than raw API

---

## 2. Mastra AI Framework

### Purpose
TypeScript agent/workflow framework that sits on top of language models and provides orchestration patterns for complex AI applications.

### Key Concepts

#### Agents
Autonomous entities with instructions, tools, and optional workflows/sub-agents.

```typescript
import { Agent } from '@mastra/core/agent';
import { openai } from '@ai-sdk/openai';

const weatherAgent = new Agent({
  name: 'weather-agent',
  description: 'Provides weather information',
  instructions: 'You are a helpful weather assistant. Use the weatherTool to fetch current weather data.',
  model: openai('gpt-4o'),
  tools: { weatherTool },
});

// Execute
const response = await weatherAgent.generate('What is the weather in San Francisco?');
console.log(response.text);
```

**Mapping to Dexter**: Replaces the multi-agent pattern (Planning, Action, Validation, Answer agents).

#### Workflows
Deterministic, step-based orchestration using DAG (directed acyclic graph) structure.

```typescript
import { createWorkflow, createStep } from '@mastra/core/workflows';

const step1 = createStep({
  id: 'fetch-data',
  inputSchema: z.object({ city: z.string() }),
  outputSchema: z.object({ data: z.any() }),
  execute: async ({ inputData }) => {
    // Fetch data logic
    return { data: [...] };
  },
});

const step2 = createStep({
  id: 'process-data',
  inputSchema: z.object({ data: z.any() }),
  outputSchema: z.object({ result: z.string() }),
  execute: async ({ inputData }) => {
    // Process data
    return { result: 'processed' };
  },
});

const workflow = createWorkflow({
  id: 'data-pipeline',
  inputSchema: z.object({ city: z.string() }),
  outputSchema: z.object({ result: z.string() }),
})
  .then(step1)
  .then(step2)
  .commit();

// Execute
const result = await workflow.execute({
  triggerData: { city: 'San Francisco' },
});
```

**Mapping to Dexter**: Could replace the task decomposition/execution loop:
- Step 1: Plan tasks (like `plan_tasks()`)
- Step 2: Execute actions (like `ask_for_actions()`)
- Step 3: Validate (like `ask_if_done()`)

#### Tools
Type-safe tool definitions with Zod schemas.

```typescript
import { createTool } from '@mastra/core/tools';

const calculatorTool = createTool({
  id: 'calculate',
  description: 'Perform mathematical calculations',
  inputSchema: z.object({
    operation: z.enum(['add', 'subtract', 'multiply', 'divide']),
    a: z.number(),
    b: z.number(),
  }),
  outputSchema: z.object({
    result: z.number(),
  }),
  execute: async ({ context }) => {
    let result;
    switch (context.operation) {
      case 'add': result = context.a + context.b; break;
      case 'subtract': result = context.a - context.b; break;
      case 'multiply': result = context.a * context.b; break;
      case 'divide': result = context.a / context.b; break;
    }
    return { result };
  },
});

// Use with agent
const agent = new Agent({
  name: 'math-agent',
  model: openai('gpt-4o'),
  tools: { calculatorTool },
});
```

**Mapping to Dexter**: Replaces `tools/__init__.py` tool registry with type-safe Mastra tool definitions.

#### Agent Networks
Multi-agent collaboration with routing and delegation.

```typescript
const agent = new Agent({
  name: 'router-agent',
  instructions: 'Route queries to appropriate agents',
  model: openai('gpt-4o'),
  agents: {
    weatherAgent,
    newsAgent,
  },
  workflows: {
    researchWorkflow,
  },
  tools: {
    searchTool,
  },
});

// Execute with routing
const result = await agent.network('Find weather and news for Tokyo');

for await (const chunk of result) {
  console.log(chunk.type);
}
```

**Mapping to Dexter**: Could implement the multi-agent coordination pattern (Planning → Action → Validation → Answer agents).

#### MCP Integration
Seamlessly connect Model Context Protocol tools.

```typescript
import { MCPClient } from '@mastra/core/mcp';

const mcpClient = new MCPClient({
  servers: [
    {
      name: 'wikipedia',
      url: 'stdio',
      command: 'npx',
      args: ['@modelcontextprotocol/server-wikipedia'],
    },
  ],
});

const agent = new Agent({
  name: 'research-agent',
  model: openai('gpt-4o'),
  tools: await mcpClient.getTools(),
});
```

**Mapping to Dexter**: Future capability to add financial data APIs as MCP servers.

#### Streaming
Built-in streaming support for agents and workflows.

```typescript
const stream = await agent.stream('What is the weather?');

stream.on('data', (chunk) => {
  console.log(chunk);
});

const text = await stream.text;
const usage = stream.usage;
```

### Advantages Over Pure Anthropic SDK
- **Orchestration**: Built-in workflow/step management
- **Multi-agent patterns**: Routing, delegation, collaboration
- **Memory/RAG**: Built-in support for conversation history and knowledge bases
- **Tools abstraction**: Simplified tool definition and integration
- **MCP support**: Native integration with Model Context Protocol
- **Evaluation**: Built-in testing/eval frameworks

### Limitations
- **Opinionated**: Forces specific patterns/structure
- **Less low-level control**: Some aspects of Claude interaction abstracted away
- **Learning curve**: More complex framework to learn upfront

---

## 3. Comparison Matrix

| Feature | Vercel AI SDK | Mastra |
|---------|---------------|--------|
| **Tool execution** | Native `generateText`, `ToolLoopAgent` | Via agents (uses AI SDK internally) |
| **Streaming** | Event-based (`textStream`, `fullStream`) | Event-based |
| **Structured outputs** | `streamObject` with Zod | Via agents |
| **Workflows** | Manual orchestration | Built-in DAG step-based |
| **Multi-agent** | Manual coordination | Built-in agent networks |
| **Memory/History** | Manual tracking | Built-in storage |
| **MCP support** | Limited | Built-in |
| **Type safety** | Full TypeScript | Full TypeScript |
| **Learning curve** | Gentle | Moderate |
| **Provider support** | 50+ providers | Via AI SDK (50+ providers) |
| **Documentation** | Excellent (2,300+ examples) | Good (extensive) |
| **Community** | Large (Vercel ecosystem) | Growing |

**Key insight**: Mastra is built on top of AI SDK, so you get AI SDK's capabilities plus Mastra's orchestration.

---

## 4. Recommended Architecture

### Option A: Vercel AI SDK Only (Minimal)
Best for: Direct agent loop migration with multi-provider flexibility.

```typescript
// Structure mirrors Python closely
- src/dexter/
  ├── agent.ts          // Main orchestration (ToolLoopAgent)
  ├── models.ts         // Provider configuration
  ├── tools/            // Tool definitions (ai/tool)
  ├── schemas.ts        // Zod schemas
  ├── prompts.ts        // System prompts
  └── cli.ts            // Interactive CLI
```

**Pros**:
- Familiar structure, gentle learning curve
- Multi-provider support
- Large ecosystem
- Easy to start, easy to optimize

**Cons**:
- Manual agent coordination
- No built-in workflows

### Option B: Mastra + Vercel AI SDK (Recommended)
Best for: Leveraging modern agent/workflow patterns with full type safety and orchestration.

```typescript
// Mastra-centric structure (built on AI SDK)
- src/
  ├── agents/           // Mastra agents (planning, action, validation, answer)
  ├── workflows/        // Mastra workflows (orchestration)
  ├── tools/            // Tool definitions (uses AI SDK's tool)
  ├── models.ts         // Model/provider configuration
  ├── cli.ts            // Interactive CLI
  └── mastra.ts         // Mastra instance setup
```

**Pros**:
- Built-in orchestration patterns
- Cleaner agent composition
- Mastra's agent networks for multi-agent coordination
- Future-ready for complex features
- Built-in evaluation framework

**Cons**:
- Larger framework to learn
- Slightly more abstraction

### Option C: Hybrid (Pragmatic)
Best for: Rapid migration with easy optimization later.

```typescript
// Use both frameworks strategically
- Core tool execution: Vercel AI SDK's ToolLoopAgent
- Complex workflows: Mastra's workflow orchestration
- Transition gradually to full Mastra as complexity grows
```

**Pros**:
- Start simple, grow as needed
- Flexible migration path
- Leverages best of both

**Cons**:
- Two frameworks to maintain temporarily

---

## 5. Migration Path Summary

### Phase 1: Setup (Foundational)
- [ ] Initialize TypeScript project structure
- [ ] Install Vercel AI SDK + provider (`@ai-sdk/anthropic`, `@ai-sdk/openai`)
- [ ] Install Mastra (Option B recommended) or skip if using Option A
- [ ] Define financial API tools with Zod schemas
- [ ] Set up environment variables for API keys

### Phase 2: Core Components
- [ ] Convert `tools/` directory to AI SDK tools
- [ ] Convert `schemas.py` to Zod schemas
- [ ] Convert multi-agent pattern to Mastra agents/networks or ToolLoopAgent
- [ ] Implement main orchestration (using Mastra workflows or manual loops)

### Phase 3: API Integration
- [ ] Migrate Financial Datasets API wrappers
- [ ] Implement stock price, filing, metrics retrieval
- [ ] Convert search tools (Google News → Anthropic Web Search Tool)
- [ ] Add fallback/retry logic with provider switching

### Phase 4: CLI & UX
**Reference: CLI_FRAMEWORK_DECISION.md for complete CLI framework standardization**
- [ ] Implement interactive CLI (prompt-toolkit → @inquirer/prompts v7)
- [ ] Add command history with file persistence (custom CommandHistory class)
- [ ] Implement progress indicators (ora v8) and colors (chalk v5)
- [ ] Add streaming response support with real-time tool monitoring

### Phase 5: Testing & Evaluation
- [ ] Port evaluation framework (use Mastra evals if Option B)
- [ ] Set up LangSmith integration or Mastra evals
- [ ] Implement correctness scoring
- [ ] Add unit tests for tools and agents

### Phase 6: Optimization & Deployment
- [ ] Profile and optimize performance
- [ ] Implement caching strategies (AI SDK middleware)
- [ ] Add monitoring/logging with structured events
- [ ] Deploy to production (Vercel, AWS Lambda, etc.)

---

## 6. Key Differences to Understand

### Agent Execution Model
**Python (Current):**
```python
# Manual loop in agent.py
while not is_goal_achieved():
    tasks = plan_tasks()
    for task in tasks:
        while not task_done():
            action = ask_for_actions()
            execute_tool()
            validate_task()
answer = generate_answer()
```

**TypeScript (Vercel AI SDK - Option A):**
```typescript
// ToolLoopAgent handles the loop automatically
import { ToolLoopAgent } from 'ai';

const agent = new ToolLoopAgent({
  model: anthropic('claude-opus-4-20250514'),
  tools: { planTool, actionTool, validationTool, answerTool },
  system: 'You are a financial analyst...',
});

const result = await agent.execute({
  input: 'User query',
});
```

**TypeScript (Mastra - Option B):**
```typescript
// Explicit workflow steps with orchestration
const workflow = createWorkflow({
  id: 'financial-research',
  inputSchema: z.object({ query: z.string() }),
  outputSchema: z.object({ answer: z.string() }),
})
  .then(planningStep)      // Decompose query
  .then(actionStep)        // Execute tools
  .then(validationStep)    // Validate completion
  .then(answerStep)        // Synthesize answer
  .commit();

const result = await workflow.execute({
  triggerData: { query: 'User query' },
});
```

### Tool Definition
**Python (Current):**
```python
from langchain.tools import tool

@tool
def get_stock_price(ticker: str):
    """Get current stock price"""
    return fetch_price(ticker)
```

**TypeScript (Vercel AI SDK - Option A):**
```typescript
import { tool } from 'ai';

const getStockPriceTool = tool({
  description: 'Get current stock price',
  inputSchema: z.object({
    ticker: z.string().describe('Stock ticker symbol'),
  }),
  execute: async ({ ticker }) => fetch_price(ticker),
});
```

**TypeScript (Mastra - Option B):**
```typescript
import { createTool } from '@mastra/core/tools';

const getStockPriceTool = createTool({
  id: 'get-stock-price',
  description: 'Get current stock price',
  inputSchema: z.object({
    ticker: z.string(),
  }),
  execute: async ({ context }) => fetch_price(context.ticker),
});
```

**Note**: Both AI SDK and Mastra use identical tool syntax. Mastra's tools internally use AI SDK.

### Type Safety
- Python: Runtime Pydantic validation
- TypeScript: Compile-time (via TypeScript) + runtime (via Zod) validation

---

## 7. Resource Requirements

### Dependencies (Option A: AI SDK Only)
```json
{
  "ai": "^7.0.0",
  "@ai-sdk/anthropic": "^0.0.50",
  "@ai-sdk/openai": "^0.0.50",
  "zod": "^3.22.0",
  "dotenv": "^16.3.1"
}
```

### Dependencies (Option B: AI SDK + Mastra)
```json
{
  "ai": "^7.0.0",
  "@mastra/core": "^0.10.0",
  "@ai-sdk/anthropic": "^0.0.50",
  "@ai-sdk/openai": "^0.0.50",
  "zod": "^3.22.0",
  "dotenv": "^16.3.1"
}
```

### Development Tools
- **TypeScript 5.0+**: Language
- **Node.js 18+**: Runtime
- **tsx** or **ts-node**: TypeScript executor
- **jest** or **vitest**: Testing framework
- **eslint**: Code linting
- **prettier**: Code formatting

---

## 8. Final Recommendation

### For Your Use Case (Dexter Financial Agent):

**Choose Option B: Mastra + Vercel AI SDK** (Recommended)

**Rationale:**
1. **Natural architecture fit**: Dexter's 4-agent system maps cleanly to Mastra agents
2. **Explicit workflows**: Planning → Action → Validation → Answer workflow structure
3. **Multi-provider support**: Use Claude for complex reasoning, fallback to GPT-4 or Gemini
4. **Ecosystem**: Largest AI/ML TypeScript ecosystem (2,300+ code examples)
5. **Future-ready**: MCP support, built-in evals, agent networks
6. **Type safety**: Full TypeScript compile-time + Zod runtime validation
7. **Streaming**: Built-in support for interactive CLI with real-time tool execution
8. **Easy migration**: Cleaner mental model than Python LangChain

**Alternative: Option A (Vercel AI SDK Only)** if you prefer:
- Simpler architecture
- Faster initial implementation
- Easier mental model for simple agents
- Start here, migrate to Mastra later if needed

---

## 9. Implementation Starting Point

### Quick Start (Option A - AI SDK)

```typescript
// src/tools/stock.ts
import { tool } from 'ai';
import { z } from 'zod';

export const getStockPrice = tool({
  description: 'Get current stock price',
  inputSchema: z.object({
    ticker: z.string(),
  }),
  execute: async ({ ticker }) => ({
    ticker,
    price: await fetchPrice(ticker),
  }),
});

// src/agent.ts
import { ToolLoopAgent } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { getStockPrice } from './tools/stock';

const agent = new ToolLoopAgent({
  model: anthropic('claude-opus-4-20250514'),
  tools: { getStockPrice },
  system: 'You are a financial analyst. Use tools to research stocks.',
});

// src/index.ts
const result = await agent.execute({
  input: 'Analyze Apple stock',
});

console.log(result.text);
```

### Quick Start (Option B - Mastra)

```typescript
// src/tools/stock.ts (same as above)
export const getStockPrice = tool({...});

// src/agents/analyzer.ts
import { Agent } from '@mastra/core/agent';
import { anthropic } from '@ai-sdk/anthropic';
import { getStockPrice } from '../tools/stock';

export const analyzerAgent = new Agent({
  name: 'stock-analyzer',
  model: anthropic('claude-opus-4-20250514'),
  instructions: 'You are a financial analyst. Use tools to research stocks.',
  tools: { getStockPrice },
});

// src/index.ts
const result = await analyzerAgent.generate('Analyze Apple stock');
console.log(result.text);
```

---

## 10. Key Takeaways

✅ **Vercel AI SDK** provides:
- Provider-agnostic abstraction (Claude, GPT-4, Gemini, etc.)
- Large ecosystem with 2,300+ examples
- Built-in `ToolLoopAgent` for autonomous tool execution
- Native streaming support
- Multi-provider fallback and cost optimization

✅ **Mastra** adds:
- Workflow orchestration (explicit DAG-based steps)
- Agent networks (multi-agent coordination)
- Built-in memory and state management
- Evaluation framework
- MCP server integration

✅ **For Dexter specifically**:
- Option B (Mastra + AI SDK) provides the clearest mental model
- Direct mapping: Python agents → Mastra agents
- Workflows replace manual loops
- Same tool syntax across both frameworks

🚀 **Next Steps**:
1. Choose your option (B recommended)
2. Set up TypeScript project
3. Start with Phase 1 (Setup)
4. Incrementally migrate components
5. Test against evaluation dataset

This document is now ready to guide your TypeScript conversion!
