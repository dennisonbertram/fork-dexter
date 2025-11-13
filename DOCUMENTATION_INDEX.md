# Dexter TypeScript Conversion - Documentation Index

**Last Updated**: 2025-11-13
**Status**: Planning Complete - Ready for Implementation

---

## Purpose

This directory contains complete planning documentation for converting Dexter (Python financial research agent) to TypeScript. All plans are final and ready for implementation. This documentation package is designed to be bundled into the new TypeScript project as the implementation guide.

---

## Core Planning Documents

### 1. TYPESCRIPT_CONVERSION_MASTER_PLAN.md
**Purpose**: Comprehensive master roadmap for the entire conversion project

**Contents**:
- Executive summary and architecture overview
- Complete dependency list
- 5 component conversion plans (Core Agent, Financial Tools, CLI/UI, Evaluation, Search)
- 24-week implementation roadmap (8 phases)
- Risk assessment and success metrics
- File structure and checklists

**When to Reference**:
- Project overview and timeline
- Phase-by-phase implementation guidance
- Architecture decisions and rationale
- Weekly task breakdowns

---

### 2. FOUNDATION_ANALYSIS.md
**Purpose**: Deep dive into the foundational libraries (Vercel AI SDK + Mastra)

**Contents**:
- Vercel AI SDK capabilities and patterns
- Mastra AI Framework features
- Detailed comparison matrix
- Code examples for all major patterns
- Migration strategy recommendations
- Recommended approach (Mastra + AI SDK hybrid)

**When to Reference**:
- Understanding AI SDK vs Mastra capabilities
- Learning tool definition patterns
- Implementing agent orchestration
- Provider switching strategies
- Streaming and structured output patterns

---

### 3. CLI_FRAMEWORK_DECISION.md
**Purpose**: Standardized CLI framework choices and implementation patterns

**Contents**:
- Selected libraries with rationale (@inquirer/prompts, chalk, ora)
- Complete dependency specifications
- Implementation guidelines and code examples
- Migration path from Python components
- Testing strategy for CLI components
- UI class structure patterns

**When to Reference**:
- Phase 1 project setup (dependency installation)
- Phase 4 CLI & UI implementation
- Any CLI-related implementation decisions
- Testing CLI components

**Key Decision**: All CLI code must reference this standardized framework stack

---

### 4. TESTING_TEMPLATES.md
**Purpose**: Comprehensive testing templates for unit, integration, and evaluation testing

**Contents**:
- Three-layer testing strategy (Unit, Integration, Evaluation)
- Evalite setup and configuration
- 9 testing templates with complete code examples
- 3 custom scorer implementations (correctness, quality, efficiency)
- CI/CD integration patterns
- Testing checklists for all phases
- Success metrics and benchmarks

**When to Reference**:
- Phase 2: Writing unit tests for tools
- Phase 3: Writing integration tests for agent workflows
- Phase 4: Writing UI component tests
- Phase 5: Setting up Evalite and creating evaluations
- Any testing-related implementation
- Creating custom scorers for LLM evaluation

**Key Features**:
- Vitest configuration for unit/integration tests
- Evalite configuration for LLM evaluations
- Template 1-3: Unit testing patterns
- Template 4-5: Integration testing patterns
- Template 6-9: Evaluation (Evalite) patterns
- Custom Scorers: Financial correctness, answer quality, tool efficiency

---

## Supporting Documentation

### Python Source Analysis
- **Location**: Original Dexter Python repository
- **Key Files Analyzed**:
  - `/src/dexter/agent.py` - Main orchestration (281 lines)
  - `/src/dexter/tools/` - 15 tools (14 financial + 1 search)
  - `/src/dexter/cli.py` - Interactive CLI
  - `/src/dexter/utils/ui.py` - Spinner, colors, UI class
  - `/src/dexter/evals/` - Evaluation framework

### Component Inventories
Detailed inventories embedded in master plan:
- Core Agent System: 4 files → 5 TypeScript files
- Financial Tools: 14 tools in 7 files → 8 TypeScript files
- CLI & UI: 4 files → 4 TypeScript files
- Evaluation: 3 files → 4 TypeScript files
- Search: 1 file → 2 TypeScript files

---

## Key Architectural Decisions

All decisions are documented with rationale. Key choices:

| Decision | Choice | Rationale | Document |
|----------|--------|-----------|----------|
| **LLM SDK** | Vercel AI SDK | Multi-provider, 2,300+ examples, larger ecosystem | FOUNDATION_ANALYSIS.md |
| **Orchestration** | Mastra + AI SDK hybrid | Best orchestration + full control | FOUNDATION_ANALYSIS.md |
| **CLI Input** | @inquirer/prompts v7 | Modern ESM-first, type-safe, 2M+ weekly downloads | CLI_FRAMEWORK_DECISION.md |
| **CLI Colors** | chalk v5 | Industry standard, 50M+ weekly downloads | CLI_FRAMEWORK_DECISION.md |
| **CLI Spinners** | ora v8 | Async-friendly, event-loop based | CLI_FRAMEWORK_DECISION.md |
| **History** | Custom CommandHistory | File-based persistence matching Python | CLI_FRAMEWORK_DECISION.md |
| **Schema Validation** | Zod | TypeScript-native Pydantic replacement | MASTER_PLAN.md |
| **Evaluation** | LangSmith + Custom | Mature dataset management | MASTER_PLAN.md |
| **Search** | Hybrid (RSS + Web Search) | Complementary strengths | MASTER_PLAN.md |
| **CSV Parsing** | csv-parse v5 | Stream-based, TypeScript support | CLI_FRAMEWORK_DECISION.md |
| **RSS Parsing** | rss-parser v3 | Promise-based, active maintenance | CLI_FRAMEWORK_DECISION.md |

---

## Complete Dependency List

**See**: TYPESCRIPT_CONVERSION_MASTER_PLAN.md § Foundation Libraries
**See**: CLI_FRAMEWORK_DECISION.md § Complete Dependency List

**Core Dependencies**:
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

**Total Size**: ~2.5MB (all dependencies combined)
**All Licenses**: MIT compatible
**Testing Stack**: Vitest (unit/integration) + Evalite (LLM evals) + Autoevals (built-in scorers)

---

## Implementation Timeline

**Total Duration**: 20-24 weeks (5-6 months)

### Phase Breakdown

| Phase | Duration | Focus | Key Deliverables |
|-------|----------|-------|------------------|
| **Phase 1** | 2 weeks | Foundation | Project skeleton, schemas, prompts |
| **Phase 2** | 5 weeks | Tools | All 15 tools functional and tested |
| **Phase 3** | 4 weeks | Core Agent | Agent runs queries end-to-end |
| **Phase 4** | 3 weeks | CLI & UI | Interactive CLI with rich UI |
| **Phase 5** | 4 weeks | Evaluation | Evaluation suite running |
| **Phase 6** | 2 weeks | Integration | System feature-complete |
| **Phase 7** | 2 weeks | Optimization | Performance tuned |
| **Phase 8** | 2 weeks | Polish | Production-ready |

**See**: TYPESCRIPT_CONVERSION_MASTER_PLAN.md § Implementation Roadmap for weekly task breakdowns

---

## Project Structure (Target TypeScript)

```
dexter-ts/
├── src/
│   ├── agent.ts              // Main agent orchestration
│   ├── model.ts              // LLM interface (AI SDK wrapper)
│   ├── schemas.ts            // Zod schemas
│   ├── prompts.ts            // System prompts
│   ├── cli.ts                // CLI entry point
│   ├── tools/
│   │   ├── finance/
│   │   │   ├── api.ts        // FinancialDatasetsClient
│   │   │   ├── fundamentals.ts
│   │   │   ├── filings.ts
│   │   │   ├── metrics.ts
│   │   │   ├── prices.ts
│   │   │   ├── news.ts
│   │   │   ├── estimates.ts
│   │   │   └── segments.ts
│   │   ├── search/
│   │   │   ├── google.ts     // Google News RSS
│   │   │   └── web.ts        // Anthropic Web Search
│   │   └── index.ts          // Tool registry
│   ├── utils/
│   │   ├── ui.ts             // Colors, Spinner, UI classes
│   │   ├── logger.ts         // Logger facade
│   │   ├── intro.ts          // Welcome screen
│   │   └── history.ts        // CommandHistory
│   └── evals/
│       ├── evaluator.ts      // LangSmith integration
│       ├── dataset.ts        // Dataset loader
│       └── data/
│           └── vals-finance-agent-50.csv
├── tests/
│   ├── tools/                // Tool unit tests
│   ├── agent/                // Agent integration tests
│   └── cli/                  // CLI tests
├── docs/
│   └── planning/             // This documentation package
├── .env.example
├── package.json
├── tsconfig.json
├── vitest.config.ts
└── README.md
```

---

## How to Use This Documentation

### For Initial Setup (Phase 1)
1. Read TYPESCRIPT_CONVERSION_MASTER_PLAN.md § Foundation Libraries
2. Reference CLI_FRAMEWORK_DECISION.md for exact dependency versions
3. Follow Phase 1 checklist week-by-week

### For Component Implementation
1. Find your phase in TYPESCRIPT_CONVERSION_MASTER_PLAN.md § Implementation Roadmap
2. Reference CLI_FRAMEWORK_DECISION.md when implementing CLI components
3. Reference FOUNDATION_ANALYSIS.md for AI SDK and Mastra patterns
4. Follow weekly checklists

### For Architecture Questions
1. Check TYPESCRIPT_CONVERSION_MASTER_PLAN.md § Key Architectural Decisions table
2. Read relevant sections in FOUNDATION_ANALYSIS.md
3. For CLI questions, always check CLI_FRAMEWORK_DECISION.md first

### For Code Examples
- **AI SDK patterns**: FOUNDATION_ANALYSIS.md
- **Mastra patterns**: FOUNDATION_ANALYSIS.md
- **CLI patterns**: CLI_FRAMEWORK_DECISION.md
- **Component examples**: TYPESCRIPT_CONVERSION_MASTER_PLAN.md (embedded in component sections)

---

## Success Criteria

**Functional Parity**:
- [ ] All 15 tools working (14 financial + 1 search)
- [ ] 4-phase agent loop operational
- [ ] Interactive CLI with history
- [ ] Rich UI (colors, spinners)
- [ ] Evaluation suite running
- [ ] Passes all 50 test cases

**Performance**:
- [ ] Within 120% of Python performance
- [ ] < 500ms median tool execution time
- [ ] < 30s median query response time

**Quality**:
- [ ] 80%+ test coverage
- [ ] Zero TypeScript errors (strict mode)
- [ ] Zero ESLint errors
- [ ] All tests passing
- [ ] Comprehensive documentation

**See**: TYPESCRIPT_CONVERSION_MASTER_PLAN.md § Success Metrics for complete criteria

---

## Critical Implementation Notes

### TDD Requirements
**ALL code must be written following Test-Driven Development (TDD)**:
1. Write failing test first (RED)
2. Write minimal code to pass (GREEN)
3. Refactor for quality (REFACTOR)

**Exception**: Configuration files, documentation, trivial changes only

### CLI Framework Consistency
**CRITICAL**: All CLI code MUST use the standardized framework:
- Interactive input: `@inquirer/prompts`
- Colors: `chalk`
- Spinners: `ora`
- History: Custom `CommandHistory` class

**Never** deviate from CLI_FRAMEWORK_DECISION.md without updating that document first.

### Build Quality Checks
Before marking any phase complete:
```bash
npm run lint && npm run type-check && npm run build && npm test
```

All checks must pass before moving to next phase.

### Provider Flexibility
All LLM calls must go through AI SDK abstraction to allow easy provider switching:
```typescript
import { anthropic } from '@ai-sdk/anthropic';
// OR
import { openai } from '@ai-sdk/openai';

// Same code works with both
const result = await generateText({
  model: anthropic('claude-sonnet-4-5'), // or openai('gpt-4')
  prompt: 'Your prompt'
});
```

---

## Next Steps

1. **Review Documentation**: Read all three core documents
2. **Approve Plans**: Confirm all architectural decisions
3. **Initialize Project**: Follow Phase 1, Week 1 checklist
4. **Begin Implementation**: Start Phase 2 (Tools) after Phase 1 complete

---

## Document Maintenance

**Update Policy**:
- Update this index when adding new planning documents
- Update master plan when implementation uncovers changes
- Update CLI framework decision if new CLI patterns emerge
- Keep all documents in sync

**Version Control**:
- All planning documents tracked in Git
- Tag releases at each phase completion
- Maintain changelog of architectural changes

---

## Contact & Support

**Documentation Author**: Claude (Anthropic)
**Creation Date**: November 13, 2025
**Conversion Lead**: [To be assigned]

**For Questions**:
- Architecture: See FOUNDATION_ANALYSIS.md
- Timeline: See TYPESCRIPT_CONVERSION_MASTER_PLAN.md
- CLI: See CLI_FRAMEWORK_DECISION.md

---

**END OF INDEX**
