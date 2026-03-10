# Promptfoo Study Plan: Zero to Hero

A systematic learning plan to deeply understand how promptfoo works — covering both theory and implementation. No time constraints. By the end, you will understand every major subsystem and be able to contribute confidently.

---

## How to Use This Plan

- Each **Phase** builds on the previous one
- Each **Module** within a phase has:
  - **Theory** (concepts to understand)
  - **Implementation** (code to read)
  - **Hands-On** (exercises to do)
  - **Checkpoint** (verify your understanding)
- Don't skip phases. The order matters.
- Take notes as you go. Understanding deepens with writing.

---

## Phase 1: Foundations (User-Level Understanding)

**Goal:** Use promptfoo as a power user before diving into code.

### Module 1.1: What Problem Does Promptfoo Solve?

**Theory:**
- LLMs are non-deterministic — the same prompt can produce different outputs
- "Vibe-based" testing (manually reading outputs) doesn't scale
- Promptfoo brings software engineering testing practices to AI:
  - Automated test cases with expected outputs
  - Regression testing (did my prompt change break something?)
  - Model comparison (which model is better for my use case?)
  - Security testing (can my AI be tricked?)

**Read:**
- The main `README.md` in the project root
- https://www.promptfoo.dev/docs/intro/

**Hands-On:**
1. Install promptfoo: `npm install -g promptfoo`
2. Run `promptfoo init --example getting-started`
3. Read the generated `promptfooconfig.yaml` carefully
4. Run `promptfoo eval` and observe the output
5. Run `promptfoo view` and explore the dashboard

**Checkpoint:** Can you explain to someone why automated LLM evaluation matters?

---

### Module 1.2: Configuration Deep Dive

**Theory:**
- YAML is a data format (like JSON but more human-readable)
- Promptfoo configs define three things: prompts, providers, and tests
- Nunjucks templates (`{{variable}}`) allow dynamic prompts
- `file://` references load external content

**Implementation — Read these example configs:**
```
examples/getting-started/promptfooconfig.yaml
examples/compare-claude-vs-gpt/promptfooconfig.yaml
examples/eval-json-output/promptfooconfig.yaml
examples/eval-rag/promptfooconfig.yaml
examples/eval-self-grading/promptfooconfig.yaml
examples/config-multi-turn/promptfooconfig.yaml
examples/eval-tool-use/promptfooconfig.yaml
examples/eval-moderation/promptfooconfig.yaml
examples/ollama/promptfooconfig.yaml
examples/eval-summarization/promptfooconfig.yaml
```

**Hands-On:**
1. Create configs for each use case in `docs/USER_QUICKSTART.md`
2. Run each one and examine the results
3. Experiment: change prompts, add test cases, try different assertions
4. Export results: `promptfoo eval -o output.json` and inspect the JSON

**Checkpoint:** Can you write a config from scratch for any evaluation scenario?

---

### Module 1.3: Assertion Types Mastery

**Theory:**
- Deterministic assertions (contains, regex, is-json) are fast and reliable
- LLM-graded assertions (llm-rubric, factuality) handle subjective quality
- RAG-specific assertions measure retrieval quality
- Assertions return `pass` (boolean) and `score` (0-1 float)
- Multiple assertions combine: ALL must pass for overall pass

**Types to understand:**
```
DETERMINISTIC:        LLM-GRADED:           RAG-SPECIFIC:
- contains            - llm-rubric          - answer-relevance
- icontains           - factuality          - context-faithfulness
- not-contains        - model-graded-closedqa - context-recall
- regex               - select-best         - context-relevance
- starts-with         - moderation
- is-json
- javascript          SIMILARITY:           META:
- python              - similar             - cost
- equals              - rouge-n             - latency
- contains-any        - bleu                - finish-reason
- contains-all        - bert-score
- levenshtein
```

**Hands-On:**
1. Create a config that uses every assertion type listed above
2. Intentionally make tests fail to understand error messages
3. Write 3 custom JavaScript assertions of increasing complexity
4. Use `llm-rubric` with different rubric criteria and observe scoring

**Checkpoint:** Given any quality requirement, can you choose the right assertion?

---

## Phase 2: JavaScript & TypeScript Fundamentals

**Goal:** Build the programming foundation needed to read the codebase.

*Skip this phase if you're already comfortable with JavaScript/TypeScript.*

### Module 2.1: JavaScript Essentials (for C/Java Developers)

**Theory — Key Differences from C/Java:**

```
Concept              C/Java                    JavaScript/TypeScript
============         ============              ====================
Variables            int x = 5;                const x: number = 5;
Functions            void foo(int x) {}        function foo(x: number): void {}
Lambdas              (x) -> x * 2             (x) => x * 2
Classes              class Foo {}              class Foo {}
Interfaces           interface Bar {}          interface Bar {} (TS only)
Null handling         null check               ?. (optional chaining)
String format        String.format()           `template ${literal}`
Collections          ArrayList<String>         string[]
Maps                 HashMap<K,V>              Map<K,V> or Record<K,V>
Async                Thread / Future           async/await / Promise
Modules              import pkg;               import { x } from './mod.js'
No header files      .h/.c split              Everything in .ts files
```

**Hands-On:**
1. Complete the JavaScript track on https://javascript.info/ (first 6 chapters)
2. Read https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html
3. Write a small TypeScript program that:
   - Reads a JSON file
   - Processes data with map/filter/reduce
   - Uses async/await to fetch a URL
   - Uses interfaces for type safety

---

### Module 2.2: Node.js & npm Essentials

**Theory:**
- Node.js = JavaScript runtime (like JVM for Java)
- npm = package manager (like Maven/Gradle)
- `package.json` = project manifest (like pom.xml)
- `node_modules/` = downloaded dependencies
- ESM vs CommonJS: two module systems (promptfoo uses ESM)
- `tsx` = tool that runs TypeScript directly without compiling

**Hands-On:**
1. Create a Node.js project from scratch:
   ```bash
   mkdir learn-node && cd learn-node
   npm init -y
   ```
2. Install a package: `npm install axios`
3. Write a script that makes an HTTP request
4. Add a script to package.json and run it with `npm run`
5. Understand `package-lock.json` (exact dependency versions)

---

### Module 2.3: Key Libraries Used in Promptfoo

**Read the docs for each (just the "Getting Started" sections):**

| Library | Used For | Doc |
|---------|----------|-----|
| Commander.js | CLI argument parsing | https://github.com/tj/commander.js |
| Express.js | HTTP server | https://expressjs.com/en/starter/hello-world.html |
| Nunjucks | Template rendering | https://mozilla.github.io/nunjucks/getting-started.html |
| Drizzle ORM | Database queries | https://orm.drizzle.team/docs/overview |
| Vitest | Testing | https://vitest.dev/guide/ |
| Biome | Linting/Formatting | https://biomejs.dev/ |

**Hands-On:**
1. Build a mini Express.js server with 2 API routes
2. Write a Nunjucks template with variables and conditionals
3. Write 5 Vitest tests with describe/it/expect

---

## Phase 3: Core Engine Deep Dive

**Goal:** Understand the evaluation pipeline in detail.

### Module 3.1: Entry Points — How It All Starts

**Theory:**
- `src/entrypoint.ts` is the binary entry point (what `promptfoo` command runs)
- `src/main.ts` sets up Commander.js with all CLI commands
- `src/index.ts` is the library API (for `import from 'promptfoo'`)
- Each command is a separate file in `src/commands/`

**Implementation — Read in this order:**
```
1. src/entrypoint.ts     (very short — just imports and runs main)
2. src/main.ts           (see how Commander.js registers commands)
3. src/commands/eval.ts   (the eval command handler)
4. src/commands/init.ts   (simpler — good for understanding pattern)
5. src/commands/view.ts   (starts the web server)
```

**Hands-On:**
1. Add `console.log` statements to trace the eval command flow
2. Run `npm run local -- eval` with your trace statements
3. Remove the console.logs when done

**Checkpoint:** Can you trace the execution from `promptfoo eval` to the evaluator?

---

### Module 3.2: Configuration Loading

**Theory:**
- Configs can be YAML, JSON, or JavaScript
- `file://` references are resolved relative to the config file
- `defaultTest` is merged into every test case
- `scenarios` allow grouping tests with shared config
- Nunjucks templates in prompts are rendered with test variables

**Implementation — Read:**
```
src/util/config.ts       (config loading and validation)
src/configTypes.ts       (TypeScript types for config objects)
src/prompts/index.ts     (prompt loading and processing)
src/util/templates.ts    (Nunjucks template rendering)
```

**Hands-On:**
1. Create a config that uses every feature:
   - Multiple prompts (inline, file, JSON)
   - Multiple providers with different configs
   - defaultTest, scenarios, and tests
   - Variables from files
   - Nunjucks conditionals in prompts
2. Add verbose logging and trace how config gets resolved

**Checkpoint:** Can you describe every step of config resolution?

---

### Module 3.3: The Evaluator (The Heart)

**Theory:**
- The evaluator creates a matrix: prompts × providers × test cases
- Each cell in the matrix = one API call + assertion grading
- API calls run concurrently (controlled by `maxConcurrency`)
- Results are cached by default (hash of prompt + provider + config)
- The evaluator handles retries, rate limiting, and progress reporting

**Implementation — Read carefully:**
```
src/evaluator.ts              (THE most important file)
src/evaluatorMetrics.ts       (metrics collection)
src/types.ts                  (EvalResult, ProviderResponse, etc.)
src/cache.ts                  (caching logic)
```

**Hands-On:**
1. Read `src/evaluator.ts` line by line. Add comments explaining each section.
2. Set up a debugger and step through an evaluation
3. Modify `maxConcurrency` and observe the effect on execution
4. Disable caching and compare execution times

**Checkpoint:** Can you explain the evaluator's main loop in your own words?

---

### Module 3.4: Assertions Deep Dive

**Theory:**
- Assertions are dispatched by type in a large switch/if-else
- Deterministic assertions run locally (fast)
- LLM-graded assertions make additional API calls (slower, cost money)
- Assertions return `GradingResult`: `{ pass, score, reason, componentResults }`
- Multiple assertions combine: final score = function of individual scores

**Implementation — Read:**
```
src/assertions.ts                      (or src/assertions/index.ts)
src/assertions/model-graded/           (LLM-as-judge implementations)
src/assertions/factuality.ts           (factuality checking)
src/assertions/answerRelevance.ts      (RAG metrics)
src/assertions/contextFaithfulness.ts  (RAG metrics)
```

**Hands-On:**
1. Add a new assertion type (e.g., `word-count` or `sentiment`)
2. Write tests for your new assertion
3. Create a config that uses your assertion
4. Run it and verify it works

**Checkpoint:** Can you implement any custom assertion type from scratch?

---

## Phase 4: Provider System

**Goal:** Understand how promptfoo talks to LLM APIs.

### Module 4.1: Provider Architecture

**Theory:**
- All providers implement the `ApiProvider` interface
- Provider IDs follow a `type:subtype:model` format
- The provider registry maps IDs to classes
- Provider config controls API parameters (temperature, max_tokens, etc.)
- Providers handle authentication, rate limiting, and error handling

**Implementation — Read:**
```
src/types.ts                  (ApiProvider interface definition)
src/providers/index.ts        (provider registry and resolution)
```

**Hands-On:**
1. Trace how `"openai:gpt-4"` gets resolved to an OpenAI provider instance
2. List all provider types by reading `src/providers/index.ts`

---

### Module 4.2: OpenAI Provider (Reference Implementation)

**Theory:**
- OpenAI is the most-used provider, making it the reference implementation
- Supports chat completions, responses API, and function calling
- Handles streaming, token counting, and cost calculation
- Supports tools/function calling

**Implementation — Read:**
```
src/providers/openai.ts       (read the entire file)
```

**Hands-On:**
1. Trace an API call from provider.callApi() to the HTTP request
2. Understand how token usage and cost are calculated
3. Understand how function/tool calls are handled

---

### Module 4.3: Other Providers & Custom Providers

**Implementation — Read at least 2 more providers:**
```
src/providers/anthropic.ts    (different API format than OpenAI)
src/providers/ollama.ts       (local model, different architecture)
src/providers/http.ts         (generic HTTP — shows flexibility)
src/providers/script.ts       (script execution provider)
```

**Hands-On:**
1. Create a custom provider in JavaScript:
   ```javascript
   // my-provider.js
   module.exports = class MyProvider {
     id() { return 'my-custom-provider'; }
     async callApi(prompt) {
       return { output: `Echo: ${prompt}` };
     }
   };
   ```
2. Use it in a config: `providers: [file://my-provider.js]`
3. Run an eval with your custom provider

**Checkpoint:** Can you add a new provider for any LLM API?

---

## Phase 5: Red Team System

**Goal:** Understand the security testing pipeline.

### Module 5.1: Red Team Concepts

**Theory:**
- Red teaming = adversarial testing to find security vulnerabilities
- Plugins generate attack payloads for specific vulnerability types
- Strategies transform payloads to bypass safety filters
- Graders evaluate whether attacks succeeded
- Reports summarize findings with severity levels

**Read:**
- https://www.promptfoo.dev/docs/red-team/
- `src/redteam/AGENTS.md` (if it exists)

---

### Module 5.2: Red Team Implementation

**Implementation — Read in this order:**
```
src/redteam/index.ts          (orchestrator — entry point)
src/redteam/plugins/          (attack type implementations)
  - Pick 3 plugin files and read them
src/redteam/strategies/       (evasion technique implementations)
  - Pick 3 strategy files and read them
src/redteam/graders/          (attack success evaluation)
  - Pick 2 grader files
src/redteam/report/           (report generation)
```

**Hands-On:**
1. Run a red team scan against a model:
   ```bash
   promptfoo redteam init
   promptfoo redteam run
   promptfoo view
   ```
2. Read the generated report and understand each vulnerability category
3. Trace the code path from `redteam run` to attack generation to grading

**Checkpoint:** Can you explain the red team pipeline and add a new plugin?

---

## Phase 6: Web UI & Server

**Goal:** Understand the full-stack web application.

### Module 6.1: React Fundamentals

*Skip if you already know React.*

**Theory:**
- React renders UI from components (functions that return JSX)
- JSX = HTML-like syntax in JavaScript
- State management via `useState` hook
- Side effects via `useEffect` hook
- Data fetching from API using `fetch()`

**Hands-On:**
1. Follow https://react.dev/learn (Quick Start)
2. Build a tiny React app with Vite:
   ```bash
   npm create vite@latest my-app -- --template react-ts
   cd my-app && npm install && npm run dev
   ```
3. Create a component that fetches and displays data from an API

---

### Module 6.2: The Express.js Server

**Implementation — Read:**
```
src/server/index.ts           (server setup, middleware, static files)
src/server/routes/            (API route definitions)
```

**Theory:**
- Express.js handles HTTP requests
- Middleware chain: CSRF → CORS → Body Parser → Route Handler
- API routes return JSON data from the database
- Static file serving delivers the built React app

**Hands-On:**
1. Start the dev server: `npm run dev:server`
2. Use `curl` or Postman to call API endpoints:
   ```bash
   curl http://localhost:3000/api/results
   curl http://localhost:3000/health
   ```
3. Trace an API request from the browser → server → database → response

---

### Module 6.3: The React Web UI

**Implementation — Read:**
```
src/app/src/main.tsx          (entry point)
src/app/src/App.tsx           (root component, routing)
src/app/src/pages/            (page-level components)
src/app/AGENTS.md             (frontend development guidelines)
```

**Hands-On:**
1. Start the frontend: `npm run dev:app`
2. Open http://localhost:5173 and use browser DevTools:
   - Network tab: observe API calls
   - React DevTools: inspect component tree
   - Console: check for errors
3. Make a small UI change and see hot reload in action

**Checkpoint:** Can you trace data from database → API → React component?

---

## Phase 7: Database & Storage

**Goal:** Understand data persistence.

### Module 7.1: SQLite & Drizzle ORM

**Theory:**
- SQLite = embedded database (single file, no server needed)
- Drizzle ORM = type-safe database queries in TypeScript
- Schema defined in code, migrations generated automatically
- Database location: `~/.promptfoo/promptfoo.db`

**Implementation — Read:**
```
src/database/schema.ts        (table definitions)
src/database/operations.ts    (CRUD operations)
drizzle.config.ts             (migration config)
drizzle/*.sql                 (migration files, read a few)
```

**Hands-On:**
1. Open the database GUI: `npm run db:studio`
2. Browse the tables and data
3. Write a query using Drizzle syntax
4. Create a migration:
   - Add a field to the schema
   - Run `npm run db:generate`
   - Inspect the generated SQL
   - Run `npm run db:migrate`
   - Revert (don't actually apply in production)

**Checkpoint:** Can you read, write, and query data using Drizzle?

---

## Phase 8: Testing & Quality

**Goal:** Master the testing infrastructure.

### Module 8.1: Test Architecture

**Implementation — Read:**
```
test/AGENTS.md                (testing guidelines)
vitest.config.ts              (test configuration)
vitest.setup.ts               (test setup/fixtures)
test/evaluator.test.ts        (core engine tests)
test/assertions/*.test.ts     (assertion tests, read 3)
test/providers/*.test.ts      (provider tests, read 2)
```

**Hands-On:**
1. Run the full test suite: `npm test`
2. Run a specific test: `npx vitest test/evaluator.test.ts`
3. Write tests for a module you explored earlier
4. Practice mocking:
   - Mock a provider's API call
   - Mock file system operations
   - Mock database operations

---

### Module 8.2: Integration Testing

**Implementation — Read:**
```
vitest.integration.config.ts  (integration test config)
test/ (look for integration test files)
```

**Hands-On:**
1. Run integration tests (requires API keys):
   ```bash
   npm run test:integration
   ```
2. Write an integration test that makes a real API call
3. Understand the difference between unit tests and integration tests

---

## Phase 9: Build System & DevOps

**Goal:** Understand the full development lifecycle.

### Module 9.1: Build Pipeline

**Implementation — Read:**
```
package.json                  (scripts section)
tsconfig.json                 (TypeScript config)
tsdown.config.ts              (bundler config)
src/app/vite.config.ts        (frontend bundler)
biome.jsonc                   (linter config)
.prettierrc.yaml              (formatter config)
.husky/pre-commit             (git hook)
```

**Hands-On:**
1. Run each build step individually and understand what it produces:
   ```bash
   npm run tsc                   # Type check (no output files)
   npx tsdown                    # Compile TS to JS
   npm run build:app             # Build React app
   npm run build                 # All three at once
   ```
2. Inspect the `dist/` directory after each step

---

### Module 9.2: CI/CD & Release

**Implementation — Read:**
```
.github/workflows/main.yml          (CI pipeline)
.github/workflows/release-please.yml (release automation)
.github/workflows/docker.yml         (Docker builds)
Dockerfile                           (container definition)
```

**Hands-On:**
1. Trace what happens when a PR is submitted
2. Understand the release process
3. Build and run the Docker container:
   ```bash
   docker build -t promptfoo .
   docker run promptfoo --version
   ```

---

## Phase 10: Advanced Topics

**Goal:** Master the remaining subsystems.

### Module 10.1: Caching System

**Read:** `src/cache.ts`
- How cache keys are generated
- Where cache is stored (`~/.cache/promptfoo/`)
- How cache invalidation works
- The `--no-cache` flag implementation

### Module 10.2: Sharing & Collaboration

**Read:** `src/share.ts`
- How results are uploaded for sharing
- The shareable URL generation

### Module 10.3: MCP (Model Context Protocol)

**Read:** Relevant MCP-related files in `src/`
- What MCP is and why it matters
- How promptfoo acts as an MCP server

### Module 10.4: Code Scanning (GitHub Action)

**Read:** `code-scan-action/`
- How the GitHub Action works
- How it scans PRs for LLM-related security issues

---

## Phase 11: Capstone Projects

**Goal:** Prove your mastery by building real features.

### Project 1: Add a New Provider
Add support for a new LLM API (e.g., Cohere, AI21, or a mock provider).
- Implement the `ApiProvider` interface
- Add tests
- Add an example config
- Add documentation

### Project 2: Add a New Assertion Type
Create a custom assertion (e.g., `sentiment`, `reading-level`, `toxicity`).
- Implement the assertion logic
- Add it to the type system
- Write comprehensive tests
- Add documentation and examples

### Project 3: Add a New Red Team Plugin
Create a plugin for a new vulnerability type.
- Study existing plugins
- Implement attack generation
- Implement grading
- Write tests and documentation

### Project 4: Improve the Web UI
Add a new feature to the dashboard (e.g., charts, export, filtering).
- Design the component
- Connect to the API
- Add tests
- Ensure responsive design

### Project 5: End-to-End Feature
Build a complete feature that touches all layers:
- CLI command
- Core engine logic
- API endpoint
- Web UI display
- Tests at every level
- Documentation

---

## Appendix: Recommended Reading Order for Key Files

If you want to read the codebase in the most logical order:

```
1.  README.md                    (what is this project?)
2.  AGENTS.md                    (coding guidelines)
3.  package.json                 (project structure, scripts)
4.  tsconfig.json                (TypeScript config)
5.  src/entrypoint.ts            (where it all begins)
6.  src/main.ts                  (CLI command registration)
7.  src/types.ts                 (core data types)
8.  src/configTypes.ts           (config data types)
9.  src/commands/eval.ts         (eval command)
10. src/evaluator.ts             (THE CORE ENGINE)
11. src/providers/index.ts       (provider registry)
12. src/providers/openai.ts      (reference provider)
13. src/assertions.ts            (assertion engine)
14. src/prompts/index.ts         (prompt loading)
15. src/util/config.ts           (config loading)
16. src/util/templates.ts        (Nunjucks templates)
17. src/cache.ts                 (caching)
18. src/database/schema.ts       (database schema)
19. src/database/operations.ts   (database operations)
20. src/server/index.ts          (web server)
21. src/app/src/App.tsx          (React UI entry)
22. src/redteam/index.ts         (red team engine)
23. src/commands/view.ts         (view command)
24. src/share.ts                 (sharing)
```

---

## Appendix: Study Milestones

Track your progress:

```
[ ] Phase 1:  Can use promptfoo as a power user
[ ] Phase 2:  Comfortable with TypeScript/Node.js/npm
[ ] Phase 3:  Understand the evaluation pipeline in code
[ ] Phase 4:  Understand the provider system
[ ] Phase 5:  Understand the red team system
[ ] Phase 6:  Understand the web UI and server
[ ] Phase 7:  Understand the database layer
[ ] Phase 8:  Can write effective tests
[ ] Phase 9:  Understand the build and release pipeline
[ ] Phase 10: Explored all advanced subsystems
[ ] Phase 11: Completed at least one capstone project

HERO STATUS: All boxes checked!
```
