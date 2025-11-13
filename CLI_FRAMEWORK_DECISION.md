# CLI Framework Decision Document

**Date**: 2025-11-13
**Status**: Proposed for Approval
**Decision Owner**: Conversion Team

---

## Executive Summary

This document standardizes the CLI framework choices for the Dexter TypeScript conversion. All implementation plans will reference these specific libraries to ensure consistency across the codebase.

---

## Selected CLI Framework Stack

### 1. Interactive Input & Prompts
**Library**: `@inquirer/prompts` v7.x
**Weekly Downloads**: 2M+
**Purpose**: Interactive question/answer flows

**Rationale**:
- Modern ESM-first design
- Individual prompt imports (tree-shakeable)
- Active maintenance by npm Inc.
- Type-safe prompt system
- Supports all interaction patterns Dexter needs

**Usage Pattern**:
```typescript
import { input, confirm } from '@inquirer/prompts';

const query = await input({
  message: 'Ask Dexter a question:'
});

const shouldContinue = await confirm({
  message: 'Continue with another query?'
});
```

**Alternative Considered**: Native `readline` - Rejected due to callback-based API and lack of built-in validation

---

### 2. Terminal Styling & Colors
**Library**: `chalk` v5.x
**Weekly Downloads**: 50M+
**Purpose**: Terminal text styling and colors

**Rationale**:
- Industry standard (used by 100K+ packages)
- ESM-native
- Excellent TypeScript support
- Composable styles
- Auto-detects color support

**Usage Pattern**:
```typescript
import chalk from 'chalk';

console.log(chalk.blue('Thinking...'));
console.log(chalk.green('✓ Task complete'));
console.log(chalk.red('✗ Error occurred'));
```

**Alternatives Considered**:
- `kleur` - Lighter but less feature-complete
- `picocolors` - Minimal API, no style chaining

---

### 3. Progress Indicators & Spinners
**Library**: `ora` v8.x
**Weekly Downloads**: 10M+
**Purpose**: Animated spinners and progress indicators

**Rationale**:
- Non-blocking async-friendly API
- Built-in spinner animations
- Color integration with chalk
- Dynamic message updates
- Persistence on stop (maintains visual history)

**Usage Pattern**:
```typescript
import ora from 'ora';

const spinner = ora({
  text: 'Planning tasks...',
  color: 'blue'
}).start();

spinner.text = 'Executing tool calls...';
spinner.succeed('Task completed');
```

**Alternatives Considered**:
- `cli-spinners` - Lower-level, requires manual animation
- `nanospinner` - Minimal, lacks persistence features

---

### 4. Command History
**Library**: Custom `CommandHistory` class
**Storage**: File-based (`~/.dexter_history`)
**Purpose**: Multi-session command history with persistence

**Rationale**:
- Python Dexter uses `prompt_toolkit` with file-based history
- No existing TypeScript library matches exact requirements
- Simple implementation (~50 lines)
- Full control over history format and limits

**Usage Pattern**:
```typescript
export class CommandHistory {
  private history: string[] = [];
  private historyFile: string;
  private maxSize: number = 1000;

  constructor(historyFile: string = '~/.dexter_history') {
    this.historyFile = expandHome(historyFile);
    this.loadHistory();
  }

  add(command: string): void {
    this.history.push(command);
    if (this.history.length > this.maxSize) {
      this.history.shift();
    }
    this.saveHistory();
  }

  getHistory(): string[] {
    return [...this.history];
  }

  private loadHistory(): void {
    if (fs.existsSync(this.historyFile)) {
      const content = fs.readFileSync(this.historyFile, 'utf-8');
      this.history = content.split('\n').filter(Boolean);
    }
  }

  private saveHistory(): void {
    fs.writeFileSync(this.historyFile, this.history.join('\n'));
  }
}
```

**Alternatives Considered**:
- `readline` native history - Session-only, no persistence
- `node-persist` - Overkill for simple list storage

---

### 5. CSV Parsing (for Evaluation Data)
**Library**: `csv-parse` v5.x
**Weekly Downloads**: 5M+
**Purpose**: Parse evaluation dataset CSV files

**Rationale**:
- Part of node-csv ecosystem
- Stream-based for large files
- Type-safe with TypeScript
- Handles quoted fields and escaping
- Used by LangSmith examples

**Usage Pattern**:
```typescript
import { parse } from 'csv-parse/sync';
import fs from 'fs';

const content = fs.readFileSync('evals.csv', 'utf-8');
const records = parse(content, {
  columns: true,
  skip_empty_lines: true,
});
```

**Alternatives Considered**:
- `papaparse` - Browser-focused
- `fast-csv` - Similar features, less adoption

---

### 6. RSS Feed Parsing (for News Search)
**Library**: `rss-parser` v3.x
**Weekly Downloads**: 1M+
**Purpose**: Parse Google News RSS feeds

**Rationale**:
- Simple, focused API
- Promise-based (async/await friendly)
- Handles all RSS/Atom variations
- Active maintenance
- TypeScript definitions included

**Usage Pattern**:
```typescript
import Parser from 'rss-parser';

const parser = new Parser();
const feed = await parser.parseURL(
  `https://news.google.com/rss/search?q=${query}`
);

const articles = feed.items.map(item => ({
  title: item.title,
  link: item.link,
  date: item.pubDate,
  source: item.source?.name,
}));
```

**Alternatives Considered**:
- `feedparser` - Unmaintained
- `rss2json` - Requires external service

---

## Complete Dependency List

```json
{
  "dependencies": {
    "@inquirer/prompts": "^7.0.0",
    "chalk": "^5.3.0",
    "ora": "^8.0.0",
    "csv-parse": "^5.5.0",
    "rss-parser": "^3.13.0"
  }
}
```

**Total Size Impact**: ~2.5MB (all dependencies combined)
**Security**: All packages have active security maintenance
**License Compatibility**: All MIT licensed

---

## Implementation Guidelines

### 1. UI Class Structure

```typescript
// src/utils/ui.ts
import chalk from 'chalk';
import ora, { Ora } from 'ora';

export class Colors {
  static readonly PRIMARY = chalk.blue;
  static readonly SUCCESS = chalk.green;
  static readonly ERROR = chalk.red;
  static readonly WARNING = chalk.yellow;
  static readonly INFO = chalk.cyan;
}

export class Spinner {
  private oraSpinner: Ora;

  constructor(message: string, color: string = 'blue') {
    this.oraSpinner = ora({
      text: message,
      color: color as any
    });
  }

  start(): void {
    this.oraSpinner.start();
  }

  updateMessage(message: string): void {
    this.oraSpinner.text = message;
  }

  stop(finalMessage: string, symbol: string = '✓'): void {
    this.oraSpinner.stopAndPersist({
      symbol,
      text: finalMessage
    });
  }

  succeed(message?: string): void {
    this.oraSpinner.succeed(message);
  }

  fail(message?: string): void {
    this.oraSpinner.fail(message);
  }
}

export class UI {
  private spinner: Spinner | null = null;

  print(message: string, color?: string): void {
    if (this.spinner) {
      this.spinner.stop(message);
      this.spinner = null;
    } else {
      console.log(color ? chalk.hex(color)(message) : message);
    }
  }

  printSuccess(message: string): void {
    this.print(Colors.SUCCESS(message));
  }

  printError(message: string): void {
    this.print(Colors.ERROR(message));
  }

  startSpinner(message: string, color: string = 'blue'): void {
    this.spinner = new Spinner(message, color);
    this.spinner.start();
  }

  updateSpinner(message: string): void {
    if (this.spinner) {
      this.spinner.updateMessage(message);
    }
  }

  stopSpinner(finalMessage: string, symbol: string = '✓'): void {
    if (this.spinner) {
      this.spinner.stop(finalMessage, symbol);
      this.spinner = null;
    }
  }
}
```

### 2. CLI Integration

```typescript
// src/cli.ts
import { input, confirm } from '@inquirer/prompts';
import { UI } from './utils/ui.js';
import { CommandHistory } from './utils/history.js';

export async function runCLI(): Promise<void> {
  const ui = new UI();
  const history = new CommandHistory();

  // Show intro
  console.log(await getIntro());

  while (true) {
    const query = await input({
      message: 'Ask Dexter a question:'
    });

    if (!query.trim()) continue;
    if (query.toLowerCase() === 'exit') break;

    history.add(query);

    ui.startSpinner('Thinking...', 'blue');

    try {
      const result = await agent.run(query);
      ui.stopSpinner('Done', '✓');
      ui.print(result);
    } catch (error) {
      ui.stopSpinner('Error occurred', '✗');
      ui.printError(error.message);
    }

    const shouldContinue = await confirm({
      message: 'Continue with another query?',
      default: true
    });

    if (!shouldContinue) break;
  }
}
```

---

## Migration Path from Python

| Python Component | TypeScript Replacement |
|-----------------|------------------------|
| `prompt_toolkit.PromptSession` | `@inquirer/prompts.input()` |
| `prompt_toolkit.FileHistory` | Custom `CommandHistory` class |
| `threading.Thread` (spinner) | `ora` (event-loop based) |
| `colored` / terminal colors | `chalk` |
| Manual ASCII art | Keep as template literal strings |

---

## Testing Strategy

### Unit Tests
- Mock `@inquirer/prompts` responses
- Test `CommandHistory` file operations with temp files
- Verify `UI` class spinner lifecycle
- Test color output with chalk methods

### Integration Tests
- Test full CLI flow with mocked stdin/stdout
- Verify history persistence across sessions
- Test spinner behavior during async operations

### Example Test:
```typescript
import { describe, it, expect, vi } from 'vitest';
import { UI } from './ui';

describe('UI', () => {
  it('should start and stop spinner with message', async () => {
    const ui = new UI();

    ui.startSpinner('Loading...');
    expect(ui['spinner']).not.toBeNull();

    ui.stopSpinner('Completed');
    expect(ui['spinner']).toBeNull();
  });
});
```

---

## Documentation Requirements

All CLI-related code must include:
1. JSDoc comments for public methods
2. Usage examples in function headers
3. Error handling patterns
4. TypeScript strict mode compliance

---

## Approval Checklist

- [ ] User approves library choices
- [ ] Dependencies added to package.json
- [ ] All planning docs updated with references
- [ ] Implementation guidelines reviewed
- [ ] Testing strategy confirmed

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-13 | Claude | Initial CLI framework standardization |

---

## Next Steps

1. **User Approval**: Review and approve this framework stack
2. **Documentation Update**: Update all planning docs to reference these choices
3. **Phase 1 Implementation**: Add dependencies in project initialization
