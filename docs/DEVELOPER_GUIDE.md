# Promptfoo Developer Guide

A step-by-step, hand-holding guide for developers who are new to full-stack web development. This guide assumes you have experience with C/C++/Java but are new to Node.js, TypeScript, React, and modern web tooling.

---

## Table of Contents

1. [Prerequisites & Concepts You Need](#1-prerequisites--concepts-you-need)
2. [Environment Setup (Step by Step)](#2-environment-setup)
3. [Understanding the Tech Stack](#3-understanding-the-tech-stack)
4. [Building the Project](#4-building-the-project)
5. [Running the Project](#5-running-the-project)
6. [Project Structure Walkthrough](#6-project-structure-walkthrough)
7. [How to Read the Code](#7-how-to-read-the-code)
8. [Making Your First Change](#8-making-your-first-change)
9. [Testing](#9-testing)
10. [Debugging](#10-debugging)
11. [Git Workflow](#11-git-workflow)
12. [Common Tasks](#12-common-tasks)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Prerequisites & Concepts You Need

### 1.1 What You Already Know (C/C++/Java) → What's Different Here

| You Know (C/Java) | Here (Node.js/TypeScript) | Key Difference |
|---|---|---|
| `main()` function | `src/main.ts` entry point | No compile step needed in dev (tsx runs TS directly) |
| Header files (.h) | Type definitions (.d.ts) | Types are optional but encouraged |
| Makefile / CMake | `package.json` scripts | `npm run build` instead of `make` |
| `#include <stdio.h>` | `import { x } from 'y'` | ES Modules (ESM) import system |
| Static typing (Java) | TypeScript | Similar! TS adds types to JavaScript |
| Compiled binary | `node dist/src/entrypoint.js` | JavaScript is interpreted/JIT compiled |
| Threads | async/await | Single-threaded with event loop |
| malloc/free | Garbage collected | Like Java, no manual memory management |
| JUnit / GTest | Vitest | Similar test structure (describe/it/expect) |
| Maven/Gradle deps | npm dependencies | `node_modules/` folder holds all dependencies |

### 1.2 Key Concepts to Understand

**Node.js** = A runtime that executes JavaScript outside the browser. Think of it as the JVM for JavaScript.

**npm** = Node Package Manager. Like Maven for Java or pip for Python. It:
- Installs dependencies (`npm install`)
- Runs scripts (`npm run build`)
- Manages the `package.json` file

**TypeScript** = JavaScript + static types. If you know Java, you'll feel at home:
```typescript
// Java:                          // TypeScript:
// String name = "hello";         const name: string = "hello";
// int count = 42;                const count: number = 42;
// List<String> items;            const items: string[] = [];
// Map<String, Object> data;      const data: Record<string, any> = {};
```

**async/await** = How JavaScript handles concurrency (like futures/promises):
```typescript
// Instead of threads, JavaScript uses an event loop.
// async/await makes asynchronous code look synchronous:

async function fetchData() {
  const response = await fetch('https://api.example.com/data');  // Non-blocking!
  const data = await response.json();                            // Waits for result
  return data;
}
```

**ESM (ECMAScript Modules)** = The modern JavaScript module system:
```typescript
// Export (like public methods in Java):
export function evaluate() { ... }
export class OpenAiProvider { ... }

// Import (like import statements in Java):
import { evaluate } from './evaluator.js';
import { OpenAiProvider } from './providers/openai.js';
```

---

## 2. Environment Setup

### Step 1: Install Node.js

Node.js is required. You need version **20.20+ or 22.22+**.

**Windows:**
1. Go to https://nodejs.org/
2. Download the LTS (Long Term Support) version
3. Run the installer, accept all defaults
4. Open a new terminal (PowerShell or Git Bash)
5. Verify: `node --version` (should show v20.x or v22.x)
6. Verify: `npm --version` (should show 10.x)

**macOS:**
```bash
# Using Homebrew (recommended):
brew install node@22

# Or download from https://nodejs.org/
```

**Linux (Ubuntu/Debian):**
```bash
# Using nvm (Node Version Manager) - RECOMMENDED:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
```

### Step 2: Install Git

If you don't have Git:
- **Windows:** Download from https://git-scm.com/ (includes Git Bash)
- **macOS:** `brew install git` or it comes with Xcode CLI tools
- **Linux:** `sudo apt install git`

### Step 3: Clone the Repository

```bash
git clone https://github.com/promptfoo/promptfoo.git
cd promptfoo
```

### Step 4: Install Dependencies

```bash
npm install
```

**What this does:**
- Reads `package.json` for the list of dependencies
- Downloads all packages into `node_modules/` folder
- Also installs dependencies for workspaces (src/app, site)
- Sets up the pre-commit hook via Husky

This may take 2-5 minutes on first run. The `node_modules/` folder will be ~500MB. This is normal for JavaScript projects.

### Step 5: Verify Setup

```bash
# Check TypeScript compiles without errors:
npm run tsc

# Run the test suite:
npm test

# Try running the CLI:
npm run local -- --version
```

If all three commands succeed, you're ready!

### Step 6: Set Up Your Editor

**VS Code (Recommended):**
1. Install VS Code from https://code.visualstudio.com/
2. Open the project: `code .`
3. Install recommended extensions (VS Code will prompt you):
   - Biome (linter)
   - Prettier (formatter)
   - YAML support
   - TypeScript/JavaScript support (built-in)

The project includes `.vscode/settings.json` and `.vscode/extensions.json` that configure your editor automatically.

---

## 3. Understanding the Tech Stack

### 3.1 The Stack at a Glance

```
+==========================================+
|        What              |    Tool        |
+==========================================+
| Language                 | TypeScript     |
| Runtime                  | Node.js        |
| Package Manager          | npm            |
| Backend Framework        | Express.js     |
| Frontend Framework       | React 19       |
| Frontend Build Tool      | Vite           |
| Backend Build Tool       | tsdown         |
| UI Component Library     | Material UI v7 |
| Database                 | SQLite         |
| Database ORM             | Drizzle        |
| Test Framework           | Vitest         |
| Linter                   | Biome          |
| Formatter                | Biome+Prettier |
| Git Hooks                | Husky          |
| Template Engine          | Nunjucks       |
| CLI Framework            | Commander.js   |
+==========================================+
```

### 3.2 How the Pieces Connect

```
Your Terminal
    |
    | $ promptfoo eval
    v
Commander.js (CLI framework)
    |
    | parses arguments
    v
TypeScript code (compiled to JavaScript by tsdown)
    |
    | runs on
    v
Node.js runtime
    |
    | uses npm packages from
    v
node_modules/ (dependencies)
    |
    | stores results in
    v
SQLite database (via Drizzle ORM)
    |
    | displayed in
    v
React Web UI (built by Vite, served by Express.js)
```

### 3.3 Quick Explanations of Key Technologies

**Express.js** (Backend Web Framework)
```
Think of it as a simplified HTTP server. It's like writing a simple web
server in Java with Spring Boot, but much lighter:

// Java (Spring Boot):                // Express.js:
// @GetMapping("/api/results")        app.get('/api/results', (req, res) => {
// public List<Result> getResults()     const results = db.getAll();
//   return db.getAll();                res.json(results);
// }                                  });
```

**React** (Frontend UI Framework)
```
React builds user interfaces from components. Each component is a
function that returns HTML-like syntax (JSX):

// A React component:
function ResultsTable({ results }) {
  return (
    <table>
      {results.map(r => (
        <tr key={r.id}>
          <td>{r.prompt}</td>
          <td>{r.output}</td>
          <td>{r.score}</td>
        </tr>
      ))}
    </table>
  );
}
```

**Vite** (Frontend Build Tool)
```
Vite takes your React/TypeScript source code and bundles it into
optimized HTML/CSS/JS that browsers can run. It also provides
hot-reload during development (changes appear instantly in browser).
```

**Drizzle ORM** (Database)
```
Drizzle lets you interact with SQLite using TypeScript instead of SQL:

// Instead of: SELECT * FROM evals WHERE id = ?
const result = db.select().from(evals).where(eq(evals.id, evalId));
```

---

## 4. Building the Project

### 4.1 Understanding the Build

The build does three things in parallel:

```
npm run build
    |
    +-- tsc --noEmit           (1) Type-check all TypeScript files
    |                              (finds type errors, produces NO output)
    |
    +-- tsdown                 (2) Compile TypeScript to JavaScript
    |                              Input:  src/**/*.ts
    |                              Output: dist/src/**/*.js
    |
    +-- npm run build:app      (3) Build the React web UI
                                   Input:  src/app/src/**/*.tsx
                                   Output: dist/src/app/ (HTML/CSS/JS)
```

### 4.2 Build Commands

```bash
# Full build (all three steps above):
npm run build

# Clean build (delete dist/ first, then build):
npm run build:clean && npm run build

# Watch mode (auto-rebuild on file changes):
npm run build:watch

# Build only the web UI:
npm run build:app

# Type-check only (no output files):
npm run tsc
```

### 4.3 Build Output

After building, the `dist/` directory contains:

```
dist/
+-- src/
    +-- entrypoint.js       # The CLI binary
    +-- main.js             # CLI setup
    +-- index.js            # Library exports
    +-- evaluator.js        # Core engine (compiled)
    +-- ...                 # All other compiled files
    +-- app/                # Built React web UI
        +-- index.html
        +-- assets/
            +-- *.js        # Bundled React code
            +-- *.css       # Bundled styles
```

---

## 5. Running the Project

### 5.1 Run as CLI (Testing Your Changes)

```bash
# Run any promptfoo command using your local build:
npm run local -- eval -c examples/getting-started/promptfooconfig.yaml

# The "--" is IMPORTANT! It separates npm's flags from promptfoo's flags.
# Without "--", flags go to npm instead of your code.

# Examples:
npm run local -- eval -c config.yaml --no-cache
npm run local -- view
npm run local -- init
npm run local -- --version
npm run local -- --help
```

### 5.2 Run the Development Server (Web UI)

```bash
# Start both backend server and frontend dev server:
npm run dev

# This starts:
#   - Express.js server on http://localhost:3000 (API)
#   - Vite dev server on http://localhost:5173 (Web UI with hot reload)

# Or start them separately:
npm run dev:server    # Just the API server (localhost:3000)
npm run dev:app       # Just the frontend (localhost:5173)
```

### 5.3 Using Environment Variables

Many features require API keys. Create a `.env` file in the project root:

```bash
# .env (NEVER commit this file!)
OPENAI_API_KEY=sk-your-key-here
ANTHROPIC_API_KEY=sk-ant-your-key-here
```

Then use it:
```bash
npm run local -- eval -c config.yaml --env-file .env --no-cache
```

---

## 6. Project Structure Walkthrough

### 6.1 Root Directory

```
promptfoo-main/
|
+-- package.json          # THE project manifest. Lists dependencies,
|                         # scripts, metadata. Like pom.xml (Maven).
|
+-- tsconfig.json         # TypeScript compiler settings.
|                         # Like javac compiler flags.
|
+-- vitest.config.ts      # Test runner configuration.
|                         # Like junit-platform.properties.
|
+-- biome.jsonc           # Linter rules. Like checkstyle.xml.
|
+-- drizzle.config.ts     # Database migration tool config.
|
+-- .husky/pre-commit     # Git hook: runs linting before each commit.
|
+-- AGENTS.md             # Coding guidelines for AI assistants.
+-- CLAUDE.md             # Points to AGENTS.md.
+-- CONTRIBUTING.md       # How to contribute.
+-- SECURITY.md           # Security policy.
```

### 6.2 Source Code (src/)

This is where ALL the application code lives:

```
src/
|
+-- main.ts               # CLI setup. Uses Commander.js to define
|                         # commands like "eval", "view", "init".
|                         # Analogous to main() in C/Java.
|
+-- entrypoint.ts         # Thin wrapper that imports and runs main.ts.
|                         # This is what `promptfoo` command executes.
|
+-- index.ts              # Library API. When someone does
|                         # `import { evaluate } from 'promptfoo'`,
|                         # this file defines what they get.
|
+-- evaluator.ts          # THE CORE FILE. Contains the evaluation
|                         # loop that runs prompts through providers
|                         # and grades outputs with assertions.
|
+-- types.ts              # TypeScript interfaces. Defines the shapes
|                         # of data objects used throughout the app.
|                         # Like Java interfaces/POJOs.
|
+-- configTypes.ts        # Types specifically for config files.
|
+-- assertions.ts         # Master assertion engine. Dispatches to
|                         # specific assertion implementations.
|
+-- cache.ts              # Caching layer to avoid duplicate LLM calls.
|
+-- logger.ts             # Logging infrastructure (debug, info, warn).
|
+-- share.ts              # Upload results to share publicly.
|
+-- cliState.ts           # Shared state across CLI commands.
```

### 6.3 Commands (src/commands/)

Each file = one CLI command:

```
src/commands/
|
+-- eval.ts               # `promptfoo eval`   - Run evaluations
+-- init.ts               # `promptfoo init`   - Create new project
+-- view.ts               # `promptfoo view`   - Open web dashboard
+-- list.ts               # `promptfoo list`   - List past evals
+-- show.ts               # `promptfoo show`   - Show eval details
+-- delete.ts             # `promptfoo delete` - Delete eval records
+-- cache.ts              # `promptfoo cache`  - Manage cache
+-- feedback.ts           # `promptfoo feedback` - Submit feedback
+-- generate/             # `promptfoo generate` - Generate data
+-- redteam/              # `promptfoo redteam`  - Security testing
```

### 6.4 Providers (src/providers/)

Each file = one LLM API integration:

```
src/providers/
|
+-- index.ts              # Provider registry. Maps provider ID strings
|                         # (like "openai:gpt-4") to provider classes.
|
+-- openai.ts             # OpenAI API (GPT models)
+-- anthropic.ts          # Anthropic API (Claude models)
+-- azureopenai.ts        # Azure-hosted OpenAI
+-- ollama.ts             # Ollama (local models)
+-- groq.ts               # Groq API
+-- mistral.ts            # Mistral AI
+-- http.ts               # Generic HTTP endpoint
+-- script.ts             # Shell script provider
+-- python.ts             # Python script provider
+-- webhook.ts            # Webhook-based provider
+-- browser.ts            # Browser automation provider
+-- google/               # Google (Gemini, Vertex AI)
+-- bedrock/              # AWS Bedrock
```

### 6.5 Web UI (src/app/)

A separate npm workspace (its own package.json, build process):

```
src/app/
|
+-- package.json          # Frontend dependencies (React, MUI, etc.)
+-- vite.config.ts        # Vite bundler configuration
+-- tsconfig.json         # Frontend TypeScript config
+-- src/
    +-- main.tsx          # React entry point
    +-- App.tsx           # Root React component
    +-- pages/            # Page-level components
    +-- components/       # Reusable UI components
    +-- hooks/            # Custom React hooks
    +-- utils/            # Frontend utilities
```

---

## 7. How to Read the Code

### 7.1 Start Here: The Evaluation Flow

The best way to understand the codebase is to trace an evaluation:

1. **Start at `src/commands/eval.ts`** — This is where `promptfoo eval` begins
2. **Follow to `src/evaluator.ts`** — This is where the actual work happens
3. **See how providers are loaded** — `src/providers/index.ts`
4. **See how assertions work** — `src/assertions.ts` or `src/assertions/index.ts`

### 7.2 Reading TypeScript (For C/Java Developers)

```typescript
// === Type Annotations (like Java generics) ===
function greet(name: string): string {          // Input: string, Output: string
  return `Hello, ${name}`;
}

// === Interfaces (like Java interfaces) ===
interface ApiProvider {
  id(): string;                                 // Method signature
  callApi(prompt: string): Promise<Response>;   // Returns a Promise (async)
  label?: string;                               // Optional property (?)
}

// === Async/Await (like Java CompletableFuture) ===
async function fetchData(): Promise<Data> {
  const response = await fetch(url);            // Suspends until complete
  return response.json();                       // Also async
}

// === Arrow Functions (like Java lambdas) ===
const add = (a: number, b: number) => a + b;   // Short function syntax
results.filter(r => r.score > 0.5);             // Like stream().filter()

// === Destructuring (no Java equivalent) ===
const { name, age } = person;                   // Extract object properties
const [first, ...rest] = array;                 // Extract array elements

// === Template Literals (like String.format) ===
const msg = `Hello ${name}, you scored ${score}`;

// === Optional Chaining (like Java Optional) ===
const city = user?.address?.city;               // Returns undefined if null
```

### 7.3 Key Patterns in This Codebase

**Provider Pattern** (Strategy Pattern):
```typescript
// All providers implement the same interface.
// The evaluator doesn't know which LLM it's calling.
interface ApiProvider {
  callApi(prompt: string, context?: CallContext): Promise<ProviderResponse>;
}

// Usage:
for (const provider of providers) {
  const result = await provider.callApi(prompt);  // Polymorphism!
}
```

**Assertion Pattern** (Command Pattern):
```typescript
// Each assertion is a function that takes output and returns pass/fail:
type AssertionResult = { pass: boolean; score: number; reason: string };

// The assertion engine dispatches based on type:
switch (assertion.type) {
  case 'contains':    return checkContains(output, assertion.value);
  case 'regex':       return checkRegex(output, assertion.value);
  case 'llm-rubric':  return checkLlmRubric(output, assertion.value);
  // ...
}
```

---

## 8. Making Your First Change

### 8.1 Example: Add a New Assertion Type

Let's say you want to add a `word-count` assertion that checks if the output has fewer than N words.

**Step 1: Add the assertion logic**

Find the assertion dispatcher (in `src/assertions.ts` or `src/assertions/index.ts`).
Add a new case:

```typescript
case 'word-count':
  const wordCount = output.split(/\s+/).length;
  const maxWords = Number(assertion.value);
  return {
    pass: wordCount <= maxWords,
    score: wordCount <= maxWords ? 1.0 : 0.0,
    reason: `Output has ${wordCount} words (max: ${maxWords})`,
  };
```

**Step 2: Add the type definition**

In `src/types.ts`, find the assertion type union and add `'word-count'`.

**Step 3: Write a test**

Create or modify a test file in `test/`:

```typescript
// test/assertions/wordCount.test.ts
import { describe, it, expect } from 'vitest';

describe('word-count assertion', () => {
  it('should pass when word count is within limit', () => {
    // Test implementation
  });

  it('should fail when word count exceeds limit', () => {
    // Test implementation
  });
});
```

**Step 4: Test your change**

```bash
# Run your specific test:
npx vitest test/assertions/wordCount.test.ts

# Run all tests:
npm test

# Test with a real config:
npm run local -- eval -c my-test-config.yaml --no-cache
```

**Step 5: Lint and format**

```bash
npm run l    # Lint changed files
npm run f    # Format changed files
```

### 8.2 Example: Add a New CLI Command

To add `promptfoo stats` command:

1. Create `src/commands/stats.ts`
2. Register it in `src/main.ts` (add `program.command('stats')...`)
3. Write tests in `test/commands/stats.test.ts`

---

## 9. Testing

### 9.1 Test Framework: Vitest

Vitest is like JUnit for TypeScript. Tests use `describe`, `it`, and `expect`:

```typescript
// test/evaluator.test.ts
import { describe, it, expect, vi } from 'vitest';

describe('evaluator', () => {
  it('should run all test cases', async () => {
    const result = await evaluate(config);
    expect(result.results).toHaveLength(3);
    expect(result.stats.successes).toBe(3);
  });

  it('should handle provider errors', async () => {
    const mockProvider = {
      callApi: vi.fn().mockRejectedValue(new Error('API down')),
    };
    // ...
  });
});
```

### 9.2 Running Tests

```bash
# Run ALL tests:
npm test

# Run a specific test file:
npx vitest test/evaluator.test.ts

# Run tests matching a pattern:
npx vitest --grep "should handle errors"

# Run tests in watch mode (re-runs on file changes):
npm run test:watch

# Run with verbose output:
npx vitest --reporter=verbose

# Run integration tests (requires API keys):
npm run test:integration

# Run frontend tests:
npm run test:app
```

### 9.3 Test File Locations

```
test/                     # Backend tests (globals enabled)
  +-- evaluator.test.ts   # Test the evaluation engine
  +-- assertions/         # Test assertion logic
  +-- providers/          # Test provider implementations
  +-- commands/           # Test CLI commands
  +-- ...

src/app/src/              # Frontend tests (explicit imports)
  +-- **/*.test.tsx       # React component tests
```

### 9.4 Mocking (For C/Java Developers)

Mocking in Vitest is like Mockito in Java:

```typescript
import { vi } from 'vitest';

// Mock a module (like Mockito.mock):
vi.mock('./providers/openai', () => ({
  OpenAiProvider: vi.fn().mockImplementation(() => ({
    callApi: vi.fn().mockResolvedValue({
      output: 'mocked response',
      tokenUsage: { total: 10 },
    }),
  })),
}));

// Mock a function:
const mockFn = vi.fn();
mockFn.mockReturnValue(42);
expect(mockFn).toHaveBeenCalledWith('arg');
```

---

## 10. Debugging

### 10.1 CLI Debugging

```bash
# Enable verbose logging:
npm run local -- eval -c config.yaml --verbose

# Or set log level:
LOG_LEVEL=debug npm run local -- eval -c config.yaml

# Debug information:
npm run local -- debug
```

### 10.2 VS Code Debugging

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Eval",
      "runtimeExecutable": "tsx",
      "args": ["src/main.ts", "eval", "-c", "config.yaml", "--no-cache"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    }
  ]
}
```

### 10.3 Common Debugging Techniques

```bash
# Bypass cache (stale results are a common issue):
npm run local -- eval -c config.yaml --no-cache

# Export results to JSON for inspection:
npm run local -- eval -c config.yaml -o output.json
# Then open output.json and inspect the results

# Check what providers are being used:
npm run local -- eval -c config.yaml --verbose 2>&1 | grep -i provider
```

---

## 11. Git Workflow

### 11.1 The Standard Process

```bash
# 1. Start from main (always get latest):
git checkout main
git pull origin main

# 2. Create a feature branch:
git checkout -b feature/add-word-count-assertion

# 3. Make your changes...
# (edit files, write tests)

# 4. Lint and format BEFORE committing:
npm run l    # Lint changed files
npm run f    # Format changed files

# 5. Stage specific files (don't use `git add .`):
git add src/assertions.ts test/assertions/wordCount.test.ts

# 6. Commit with conventional format:
git commit -m "feat(assertions): add word-count assertion type"

# 7. Push and create PR:
git push -u origin feature/add-word-count-assertion
```

### 11.2 Commit Message Format

```
type(scope): description

Types:
  feat     - New feature
  fix      - Bug fix
  chore    - Maintenance (deps, config)
  docs     - Documentation only
  test     - Adding/fixing tests
  refactor - Code change that neither fixes nor adds
  perf     - Performance improvement
  ci       - CI/CD changes

Scope (optional):
  eval, providers, assertions, redteam, app, server, cli, etc.

Examples:
  feat(providers): add Groq provider support
  fix(eval): handle timeout errors in concurrent evaluation
  docs: update developer guide
  test(assertions): add edge case tests for regex assertions
```

### 11.3 Rules

- **NEVER** commit/push directly to `main`
- **NEVER** use `--force` without explicit approval
- **NEVER** amend, squash, or rebase unless explicitly asked
- **ALWAYS** create a new branch for changes
- **ALWAYS** lint and format before committing

---

## 12. Common Tasks

### 12.1 Adding a New LLM Provider

1. Create `src/providers/myprovider.ts`
2. Implement the `ApiProvider` interface
3. Register in `src/providers/index.ts`
4. Add tests in `test/providers/myprovider.test.ts`
5. Add example in `examples/myprovider/`
6. Add documentation in `site/docs/providers/myprovider.md`

### 12.2 Adding a Database Migration

```bash
# 1. Modify the schema in src/database/schema.ts
# 2. Generate migration SQL:
npm run db:generate

# 3. Review the generated SQL in drizzle/NNNN_name.sql
# 4. Apply the migration:
npm run db:migrate
```

### 12.3 Working on the Web UI

```bash
# Start the frontend dev server with hot reload:
npm run dev:app
# Open http://localhost:5173

# The web UI is in src/app/src/
# Changes to .tsx files auto-reload in the browser
```

### 12.4 Updating Dependencies

**Read `docs/agents/dependency-management.md` first!**

Renovate bot manages most dependency updates automatically. Manual updates should only be done for urgent security fixes.

---

## 13. Troubleshooting

### Problem: `npm install` fails

```bash
# Try clearing npm cache:
npm cache clean --force
rm -rf node_modules
npm install

# If on Windows, try running as Administrator
# If version mismatch, use nvm:
nvm install 22
nvm use 22
```

### Problem: TypeScript errors during build

```bash
# Check errors:
npm run tsc

# Common fix: missing types
npm install @types/missing-package --save-dev
```

### Problem: Tests fail with import errors

Make sure you're using the correct import syntax. This project uses ESM:
```typescript
// Correct:
import { something } from './module.js';  // Note the .js extension

// Incorrect:
const something = require('./module');     // This is CommonJS, not ESM
```

### Problem: Cached results are stale

```bash
# Always use --no-cache during development:
npm run local -- eval -c config.yaml --no-cache
```

### Problem: Port 3000 already in use

```bash
# Find and kill the process:
# Linux/macOS:
lsof -ti:3000 | xargs kill
# Windows:
netstat -ano | findstr :3000
taskkill /PID <pid> /F
```

### Problem: Pre-commit hook fails

The pre-commit hook runs Biome linter and Prettier on staged files. Fix by:
```bash
npm run l    # Fix lint errors
npm run f    # Fix formatting
git add .    # Re-stage fixed files
git commit   # Try again
```

---

## Appendix: Key npm Scripts Reference

| Script | What It Does |
|--------|-------------|
| `npm run build` | Full build (types + compile + web UI) |
| `npm run build:clean` | Delete dist/ folder |
| `npm run build:watch` | Auto-rebuild on changes |
| `npm test` | Run all Vitest tests |
| `npm run test:watch` | Run tests, re-run on changes |
| `npm run dev` | Start dev server + frontend |
| `npm run dev:server` | Start only backend server |
| `npm run dev:app` | Start only frontend (hot reload) |
| `npm run local -- <cmd>` | Run CLI with local build |
| `npm run l` | Lint changed files only |
| `npm run f` | Format changed files only |
| `npm run lint` | Lint all src/ files |
| `npm run format` | Format all files |
| `npm run tsc` | Type-check (no output) |
| `npm run db:generate` | Generate DB migration |
| `npm run db:migrate` | Apply DB migrations |
| `npm run db:studio` | Open DB GUI |
