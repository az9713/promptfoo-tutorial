# Promptfoo Architecture Guide

A comprehensive, self-contained guide to the internal architecture of promptfoo — the open-source LLM evaluation and red-teaming framework.

---

## Table of Contents

1. [High-Level Overview](#1-high-level-overview)
2. [System Architecture Diagram](#2-system-architecture-diagram)
3. [Core Subsystems](#3-core-subsystems)
4. [The Evaluation Pipeline (Deep Dive)](#4-the-evaluation-pipeline)
5. [Provider System](#5-provider-system)
6. [Assertion & Grading System](#6-assertion--grading-system)
7. [Red Team System](#7-red-team-system)
8. [Web UI Architecture](#8-web-ui-architecture)
9. [Server Architecture](#9-server-architecture)
10. [Database Layer](#10-database-layer)
11. [CLI Architecture](#11-cli-architecture)
12. [Configuration System](#12-configuration-system)
13. [Communication Flow Diagrams](#13-communication-flow-diagrams)
14. [Build System & Toolchain](#14-build-system--toolchain)
15. [Key Files Reference](#15-key-files-reference)

---

## 1. High-Level Overview

Promptfoo is a **CLI tool and Node.js library** that evaluates LLM (Large Language Model) applications. Think of it as a "test framework" specifically designed for AI — just like JUnit tests Java code, promptfoo tests AI prompts and models.

### What It Does (Conceptually)

```
+-------------------+     +-------------------+     +-------------------+
|                   |     |                   |     |                   |
|   Your Prompts    | --> |   LLM Providers   | --> |   Assertions      |
|   (templates)     |     |   (OpenAI, etc.)  |     |   (test checks)   |
|                   |     |                   |     |                   |
+-------------------+     +-------------------+     +-------------------+
        |                         |                         |
        v                         v                         v
+-------------------------------------------------------------------+
|                                                                   |
|                    Evaluation Results                              |
|            (pass/fail, scores, comparisons)                       |
|                                                                   |
+-------------------------------------------------------------------+
```

### The Three Main User Workflows

1. **Eval (Evaluation)**: Test prompts against models with assertions
2. **Red Team**: Automatically attack AI systems to find security vulnerabilities
3. **View**: Browse results in a web-based dashboard

---

## 2. System Architecture Diagram

### Level 0: The Entire System

```
+========================================================================+
|                          PROMPTFOO SYSTEM                               |
|                                                                         |
|  +------------------+    +------------------+    +------------------+   |
|  |                  |    |                  |    |                  |   |
|  |    CLI Layer     |    |   Web UI Layer   |    |  Node.js API     |   |
|  |  (Commander.js)  |    |  (React + Vite)  |    |  (Library)       |   |
|  |                  |    |                  |    |                  |   |
|  +--------+---------+    +--------+---------+    +--------+---------+   |
|           |                       |                       |             |
|           v                       v                       v             |
|  +----------------------------------------------------------------+    |
|  |                     CORE ENGINE                                 |    |
|  |                                                                 |    |
|  |  +----------------+  +----------------+  +----------------+     |    |
|  |  | Config Loader  |  |   Evaluator    |  |  Red Team      |     |    |
|  |  | (YAML/JSON)    |  | (Test Runner)  |  |  (Attacker)    |     |    |
|  |  +----------------+  +----------------+  +----------------+     |    |
|  |                                                                 |    |
|  |  +----------------+  +----------------+  +----------------+     |    |
|  |  |   Providers    |  |  Assertions    |  |   Prompts      |     |    |
|  |  | (LLM APIs)     |  |  (Graders)     |  |  (Templates)   |     |    |
|  |  +----------------+  +----------------+  +----------------+     |    |
|  |                                                                 |    |
|  +----------------------------------------------------------------+    |
|           |                                                             |
|           v                                                             |
|  +------------------+    +------------------+                           |
|  |   SQLite DB      |    |   File System    |                           |
|  |  (Drizzle ORM)   |    | (Cache, Configs) |                           |
|  +------------------+    +------------------+                           |
|                                                                         |
+=========================================================================+
```

### Level 1: Component Interactions

```
+-------------------------------------------------------------------+
|                        USER INTERFACE LAYER                        |
|                                                                    |
|  Terminal (CLI)              Browser (Web UI)                      |
|  +----------------+         +--------------------+                 |
|  | promptfoo eval |         |  React SPA         |                 |
|  | promptfoo view |         |  localhost:5173     |                 |
|  | promptfoo init |         |  (dev) or :3000     |                 |
|  +-------+--------+         +--------+-----------+                 |
|          |                            |                            |
+----------+----------------------------+----------------------------+
           |                            |
           v                            v
+-------------------------------------------------------------------+
|                         SERVER LAYER                               |
|                                                                    |
|  +----------------------------------------------------------+     |
|  |  Express.js Server (src/server/)                          |     |
|  |                                                           |     |
|  |  /api/eval/*      - Evaluation management                 |     |
|  |  /api/results/*   - Results retrieval                     |     |
|  |  /api/prompts/*   - Prompt management                     |     |
|  |  /api/providers/* - Provider configuration                |     |
|  |  /api/redteam/*   - Red team operations                   |     |
|  |  Static files     - Serves built React app                |     |
|  +----------------------------------------------------------+     |
|                                                                    |
+-------------------------------------------------------------------+
           |
           v
+-------------------------------------------------------------------+
|                         CORE ENGINE LAYER                          |
|                                                                    |
|  Config       Evaluator      Providers     Assertions    RedTeam   |
|  Loader       Engine         Manager       Engine        Engine    |
|                                                                    |
|  YAML/JSON -> Prompts x   -> Call LLMs  -> Check       -> Attack   |
|  parsing      Test Cases      (parallel)    results       strategies|
|               = Results                     (grade)                 |
+-------------------------------------------------------------------+
           |
           v
+-------------------------------------------------------------------+
|                        DATA LAYER                                  |
|                                                                    |
|  +-------------------+  +-------------------+  +----------------+  |
|  | SQLite Database   |  |   Cache Layer     |  |  File System   |  |
|  | ~/.promptfoo/     |  | ~/.cache/         |  | Config files   |  |
|  | promptfoo.db      |  | promptfoo/        |  | Prompt files   |  |
|  +-------------------+  +-------------------+  +----------------+  |
|                                                                    |
+-------------------------------------------------------------------+
```

---

## 3. Core Subsystems

### 3.1 Directory Structure Map

```
promptfoo-main/
|
+-- src/                        # All source code
|   +-- main.ts                 # CLI entry point (tsx src/main.ts)
|   +-- entrypoint.ts           # Published binary entry point
|   +-- index.ts                # Library API exports
|   +-- evaluator.ts            # THE CORE: runs evaluations
|   +-- evaluatorMetrics.ts     # Metrics collection during eval
|   +-- types.ts                # Core TypeScript interfaces
|   +-- configTypes.ts          # Configuration type definitions
|   +-- assertions.ts           # Assertion (grading) logic
|   +-- cliState.ts             # Shared CLI state
|   +-- cache.ts                # Result caching
|   +-- logger.ts               # Logging infrastructure
|   +-- migrate.ts              # Database migration runner
|   +-- share.ts                # Result sharing functionality
|   |
|   +-- commands/               # CLI command implementations
|   |   +-- eval.ts             # `promptfoo eval` command
|   |   +-- init.ts             # `promptfoo init` command
|   |   +-- view.ts             # `promptfoo view` command
|   |   +-- generate/           # `promptfoo generate` commands
|   |   +-- redteam/            # `promptfoo redteam` commands
|   |   +-- list.ts             # `promptfoo list` command
|   |   +-- show.ts             # `promptfoo show` command
|   |   +-- delete.ts           # `promptfoo delete` command
|   |   +-- cache.ts            # `promptfoo cache` command
|   |   +-- feedback.ts         # `promptfoo feedback` command
|   |
|   +-- providers/              # LLM provider implementations
|   |   +-- openai.ts           # OpenAI (GPT models)
|   |   +-- anthropic.ts        # Anthropic (Claude models)
|   |   +-- google/             # Google (Gemini, Vertex AI)
|   |   +-- azureopenai.ts      # Azure OpenAI
|   |   +-- bedrock/            # AWS Bedrock
|   |   +-- ollama.ts           # Ollama (local models)
|   |   +-- groq.ts             # Groq
|   |   +-- mistral.ts          # Mistral AI
|   |   +-- http.ts             # Generic HTTP provider
|   |   +-- custom-api.ts       # Custom API provider
|   |   +-- script.ts           # Script-based provider
|   |   +-- python.ts           # Python provider
|   |   +-- webhook.ts          # Webhook provider
|   |   +-- browser.ts          # Browser automation
|   |   +-- index.ts            # Provider registry & loading
|   |
|   +-- assertions/             # Assertion implementations
|   |   +-- index.ts            # Assertion dispatcher
|   |   +-- model-graded/       # LLM-as-judge assertions
|   |   +-- factuality.ts       # Factuality checking
|   |   +-- moderation.ts       # Content moderation
|   |   +-- answerRelevance.ts   # RAG answer relevance
|   |   +-- contextFaithfulness.ts  # RAG context faithfulness
|   |
|   +-- prompts/                # Prompt loading & processing
|   |   +-- index.ts            # Prompt loader
|   |   +-- processors/         # Format-specific processors
|   |
|   +-- redteam/                # Red teaming / security testing
|   |   +-- index.ts            # Red team orchestrator
|   |   +-- plugins/            # Attack plugins
|   |   +-- strategies/         # Attack strategies
|   |   +-- graders/            # Red team graders
|   |   +-- report/             # Report generation
|   |
|   +-- server/                 # Express.js backend server
|   |   +-- index.ts            # Server entry point
|   |   +-- routes/             # API route definitions
|   |
|   +-- app/                    # React web UI (separate workspace)
|   |   +-- src/                # React source code
|   |   +-- package.json        # Frontend dependencies
|   |   +-- vite.config.ts      # Vite bundler config
|   |
|   +-- database/               # Database schema & operations
|   |   +-- schema.ts           # Drizzle ORM schema definitions
|   |   +-- operations.ts       # CRUD operations
|   |
|   +-- util/                   # Shared utilities
|       +-- config.ts           # Config file loading utilities
|       +-- templates.ts        # Nunjucks template processing
|       +-- transform.ts        # Output transforms
|       +-- fetch.ts            # HTTP fetch utilities
|       +-- json.ts             # JSON utilities
|
+-- test/                       # Test files (mirrors src/ structure)
+-- examples/                   # Example configurations
+-- drizzle/                    # Database migration SQL files
+-- site/                       # Documentation website (Docusaurus)
+-- docs/                       # Internal developer documentation
+-- code-scan-action/           # GitHub Action for code scanning
```

### 3.2 Subsystem Responsibilities

```
+--------------------+---------------------------------------------+
|    Subsystem       |              Responsibility                  |
+--------------------+---------------------------------------------+
| CLI (commands/)    | Parse user commands, orchestrate workflows   |
| Config Loader      | Read YAML/JSON configs, validate, resolve    |
| Evaluator          | Run prompts x providers x test cases          |
| Providers          | Abstract LLM API calls (OpenAI, Claude, etc) |
| Assertions         | Grade outputs (contains, regex, LLM-judge)   |
| Prompts            | Load & template prompts (Nunjucks)            |
| Red Team           | Generate adversarial attacks, run, grade      |
| Server             | REST API for web UI, serves static assets     |
| Web UI             | React dashboard for viewing results           |
| Database           | SQLite storage via Drizzle ORM                |
| Cache              | Avoid redundant LLM calls                    |
+--------------------+---------------------------------------------+
```

---

## 4. The Evaluation Pipeline

This is the heart of promptfoo. Understanding this flow is the key to understanding everything.

### 4.1 Conceptual Overview

An evaluation is a **matrix multiplication** of prompts, providers, and test cases:

```
                    Provider A         Provider B
                    (GPT-4)            (Claude)
                +---------------+  +---------------+
Prompt 1        |  Test 1 -> R  |  |  Test 1 -> R  |
"Translate      |  Test 2 -> R  |  |  Test 2 -> R  |
 {{input}}"     |  Test 3 -> R  |  |  Test 3 -> R  |
                +---------------+  +---------------+
Prompt 2        |  Test 1 -> R  |  |  Test 1 -> R  |
"You are a      |  Test 2 -> R  |  |  Test 2 -> R  |
 translator..." |  Test 3 -> R  |  |  Test 3 -> R  |
                +---------------+  +---------------+

R = Result (LLM output + assertion grades + score)
```

### 4.2 Detailed Pipeline Flow

```
Step 1: CONFIGURATION LOADING
==============================
User creates promptfooconfig.yaml
            |
            v
+-------------------------+
| Config Loader           |
| (src/util/config.ts)    |
|                         |
| - Read YAML file        |
| - Resolve file://refs   |
| - Apply defaultTest     |
| - Validate schema       |
| - Merge scenarios       |
+-------------------------+
            |
            v
Step 2: PROMPT RESOLUTION
==============================
+-------------------------+
| Prompt Processor        |
| (src/prompts/)          |
|                         |
| - Load from file/string |
| - Support formats:      |
|   * Plain text           |
|   * JSON (chat format)  |
|   * YAML (chat format)  |
|   * JavaScript function |
|   * Python function     |
+-------------------------+
            |
            v
Step 3: PROVIDER INITIALIZATION
================================
+-------------------------+
| Provider Manager        |
| (src/providers/index.ts)|
|                         |
| - Parse provider IDs    |
|   e.g. "openai:gpt-4"  |
| - Load provider class   |
| - Apply provider config |
| - Set up API keys       |
+-------------------------+
            |
            v
Step 4: TEST CASE EXPANSION
============================
+-------------------------+
| Test Case Builder       |
|                         |
| For each combination:   |
|   prompt x provider x   |
|   test case             |
| - Apply Nunjucks vars   |
| - Expand {{variables}}  |
| - Create API request    |
+-------------------------+
            |
            v
Step 5: LLM EXECUTION (PARALLEL)
==================================
+-------------------------+
| Evaluator Engine        |
| (src/evaluator.ts)      |
|                         |
| - Concurrent API calls  |
| - Rate limiting         |
| - Retry logic           |
| - Caching (optional)    |
| - Progress tracking     |
+-------------------------+
     |    |    |    |
     v    v    v    v
+-----+ +-----+ +-----+ +-----+
|LLM 1| |LLM 2| |LLM 3| |LLM 4|   (Parallel API calls)
+-----+ +-----+ +-----+ +-----+
     |    |    |    |
     v    v    v    v
+-------------------------+
| Raw LLM Outputs         |
| (text responses)        |
+-------------------------+
            |
            v
Step 6: ASSERTION GRADING
===========================
+-------------------------+
| Assertion Engine        |
| (src/assertions/)       |
|                         |
| For each output:        |
| - Run all assertions    |
| - Calculate scores      |
| - Determine pass/fail   |
|                         |
| Assertion types:        |
| - contains/icontains    |
| - regex                 |
| - is-json               |
| - javascript            |
| - python                |
| - llm-rubric            |
| - factuality            |
| - answer-relevance      |
| - cost / latency        |
| - moderation            |
| - ...and 40+ more       |
+-------------------------+
            |
            v
Step 7: RESULTS AGGREGATION
============================
+-------------------------+
| Results Collector       |
|                         |
| - Aggregate scores      |
| - Calculate pass rates  |
| - Compute cost totals   |
| - Generate summary      |
+-------------------------+
            |
            v
Step 8: OUTPUT & STORAGE
=========================
+-------------------------+     +-------------------------+
| Console Output          |     | SQLite Database          |
| (CLI table/progress)    |     | (Persistent storage)    |
+-------------------------+     +-------------------------+
            |                              |
            v                              v
+-------------------------+     +-------------------------+
| File Export             |     | Web UI Dashboard        |
| (JSON/CSV/HTML)         |     | (via API server)        |
+-------------------------+     +-------------------------+
```

### 4.3 Data Flow Through the Pipeline

```
promptfooconfig.yaml
    |
    |  [Parse & Validate]
    v
UnifiedConfig {
  prompts: Prompt[],
  providers: ApiProvider[],
  tests: TestCase[],
  defaultTest: TestCase,
  ...
}
    |
    |  [Expand Combinations]
    v
EvalJob[] = [
  { prompt: "...", provider: OpenAI, vars: {language: "French", input: "Hello"} },
  { prompt: "...", provider: Claude, vars: {language: "French", input: "Hello"} },
  { prompt: "...", provider: OpenAI, vars: {language: "Spanish", input: "Hello"} },
  ...
]
    |
    |  [Execute in Parallel]
    v
ProviderResponse[] = [
  { output: "Bonjour le monde", tokenUsage: {...}, cost: 0.001 },
  { output: "Bonjour tout le monde", tokenUsage: {...}, cost: 0.002 },
  ...
]
    |
    |  [Grade with Assertions]
    v
EvalResult[] = [
  {
    success: true,
    score: 0.85,
    namedScores: { relevance: 0.9, factuality: 0.8 },
    gradingResult: {
      pass: true,
      componentResults: [
        { assertion: "contains 'Bonjour'", pass: true, score: 1.0 },
        { assertion: "llm-rubric: accurate", pass: true, score: 0.7 },
      ]
    }
  },
  ...
]
    |
    |  [Aggregate & Store]
    v
EvalRecord {
  id: "eval-abc123",
  results: { table: [...], stats: {...} },
  config: { ... },
  createdAt: 1709923456
}
```

---

## 5. Provider System

### 5.1 Provider Architecture

Providers are the abstraction layer between promptfoo and any LLM API. They implement a common interface:

```
                     ApiProvider Interface
                     =====================
                     + id(): string
                     + callApi(prompt, context): ProviderResponse
                     + label?: string
                     + config?: ProviderConfig
                            ^
                            |
            +---------------+---------------+
            |               |               |
     +-----------+   +-----------+   +-----------+
     |  OpenAI   |   | Anthropic |   |  Ollama   |
     |  Provider |   |  Provider |   |  Provider |   ... 30+ providers
     +-----------+   +-----------+   +-----------+
```

### 5.2 Provider Resolution Flow

```
User writes:                    System resolves to:
==============                  ====================

"openai:gpt-4"           -->    OpenAiChatCompletionProvider
                                  model: "gpt-4"

"openai:chat:gpt-5-mini" -->    OpenAiChatCompletionProvider
                                  model: "gpt-5-mini"

"openai:responses:gpt-5" -->    OpenAiResponsesProvider
                                  model: "gpt-5"

"anthropic:messages:      -->    AnthropicMessagesProvider
 claude-sonnet-4-6"               model: "claude-sonnet-4-6"

"ollama:llama3"           -->    OllamaProvider
                                  model: "llama3"

"http://my-api/v1"        -->    HttpProvider
                                  url: "http://my-api/v1"

"exec:python my_llm.py"  -->    ScriptProvider
                                  script: "python my_llm.py"

"file://provider.js"      -->    CustomApiProvider
                                  module: "provider.js"
```

### 5.3 Provider ID Format

```
provider_id = "provider_type:sub_type:model_name"

Examples:
  openai:chat:gpt-4           (type=openai, sub=chat, model=gpt-4)
  anthropic:messages:claude-3  (type=anthropic, sub=messages, model=claude-3)
  ollama:llama3                (type=ollama, model=llama3)
  groq:mixtral-8x7b           (type=groq, model=mixtral-8x7b)

The provider registry in src/providers/index.ts parses these IDs
and instantiates the appropriate provider class.
```

### 5.4 Provider Communication

```
+------------------+      +------------------+      +------------------+
|   Evaluator      | ---> |   Provider       | ---> |   LLM API        |
|                  |      |   Instance       |      |   (External)     |
|  callApi(prompt, |      |                  |      |                  |
|   context)       |      | - Format request |      | POST /v1/chat/   |
|                  | <--- | - Handle response| <--- |   completions    |
|  ProviderResponse|      | - Extract tokens |      |                  |
|  {output, cost,  |      | - Calculate cost |      | { choices: [...] |
|   tokenUsage}    |      | - Handle errors  |      |   usage: {...} } |
+------------------+      +------------------+      +------------------+
```

---

## 6. Assertion & Grading System

### 6.1 Assertion Types Overview

```
+==================================================================+
|                    ASSERTION CATEGORIES                            |
+==================================================================+
|                                                                   |
|  DETERMINISTIC (Fast, No LLM)     LLM-GRADED (Uses LLM)          |
|  ============================     ==========================      |
|  contains / icontains             llm-rubric                      |
|  not-contains                     factuality                      |
|  starts-with                      answer-relevance                |
|  regex                            context-faithfulness             |
|  is-json                          context-recall                  |
|  is-valid-openai-tools-call       context-relevance               |
|  javascript                       model-graded-closedqa           |
|  python                           select-best                     |
|  contains-any / contains-all      moderation                      |
|  equals / not-equals                                              |
|  levenshtein                      COMPOSITE                       |
|  cost                             ==========================      |
|  latency                          human (manual review)           |
|  perplexity                       similar (embedding cosine)      |
|  finish-reason                    classifier                      |
|  not-regex                        rouge-n                         |
|  webhook                          bleu                            |
|                                   bert-score                      |
+==================================================================+
```

### 6.2 Assertion Processing Flow

```
For each LLM output:
=====================

LLM Output: "Bonjour le monde"
                |
                v
+-----------------------------------+
| Assertion 1: contains "Bonjour"   |  --> PASS (score: 1.0)
+-----------------------------------+
                |
                v
+-----------------------------------+
| Assertion 2: llm-rubric           |  --> PASS (score: 0.8)
|   "Is this a correct translation" |
|   (calls grading LLM internally) |
+-----------------------------------+
                |
                v
+-----------------------------------+
| Assertion 3: cost < 0.01          |  --> PASS (score: 1.0)
+-----------------------------------+
                |
                v
+-----------------------------------+
| Aggregate:                        |
|   All passed? -> overall PASS     |
|   Score = avg(1.0, 0.8, 1.0)     |
|         = 0.93                    |
+-----------------------------------+
```

### 6.3 LLM-as-Judge Pattern

```
+-----------------+     +------------------+     +------------------+
|  Original       |     |   Grading LLM    |     |  Grading Result  |
|  LLM Output     | --> |  (separate call) | --> |                  |
|  "Bonjour..."   |     |                  |     |  pass: true      |
|                 |     |  "Does this      |     |  score: 0.8      |
|                 |     |   output..."     |     |  reason: "The    |
+-----------------+     +------------------+     |   translation    |
                                                 |   is correct"    |
                                                 +------------------+

The grading LLM is typically a different model (or the same model)
that evaluates the quality of another model's output. This is the
"LLM-as-judge" pattern used by llm-rubric, factuality, and
RAG-specific assertions.
```

---

## 7. Red Team System

### 7.1 Red Team Overview

The red team system automatically generates adversarial attacks against your LLM application to find security vulnerabilities.

```
+========================================================================+
|                         RED TEAM PIPELINE                               |
|                                                                         |
|  Step 1            Step 2            Step 3            Step 4           |
|  =========         =========         =========         =========       |
|                                                                         |
|  +----------+     +-----------+     +-----------+     +-----------+    |
|  |  Plugin   |     | Strategy  |     | Execute   |     | Grade &   |    |
|  |  Select   | --> | Apply     | --> | Attacks   | --> | Report    |    |
|  |          |     |           |     |           |     |           |    |
|  | Choose   |     | Transform |     | Run each  |     | Evaluate  |    |
|  | attack   |     | attacks   |     | attack    |     | if attack |    |
|  | types    |     | using     |     | against   |     | succeeded |    |
|  |          |     | strategies|     | target    |     |           |    |
|  +----------+     +-----------+     +-----------+     +-----------+    |
|                                                                         |
+=========================================================================+
```

### 7.2 Plugins vs Strategies

```
PLUGINS (What to attack)              STRATEGIES (How to attack)
=========================              ==========================

+------------------+                   +------------------+
| prompt-injection |                   | jailbreak        |
| pii              |                   | crescendo        |
| harmful:hate     |                   | multi-turn       |
| harmful:violence |                   | base64           |
| overreliance     |                   | leetspeak        |
| contracts        |                   | rot13            |
| hallucination    |                   | prompt-injection |
| excessive-agency |                   | multilingual     |
| competitors      |                   | citation         |
| politics         |                   | math-prompt      |
| religion         |                   | goat             |
| ...20+ more      |                   | ...and more      |
+------------------+                   +------------------+

Plugins GENERATE attack payloads      Strategies TRANSFORM payloads
for specific vulnerability types.     to bypass safety filters.

Example:                              Example:
  Plugin "pii" generates:               Strategy "base64" transforms:
  "What is John's SSN?"                 "What is John's SSN?" -->
                                        "V2hhdCBpcyBKb2huJ3MgU1NOPw=="
```

### 7.3 Red Team Execution Flow

```
redteamConfig.yaml
    |
    v
+--------------------+
| Read target config |
| - target provider  |
| - plugins list     |
| - strategies list  |
| - numTests         |
+--------------------+
    |
    v
+--------------------+     +--------------------+
| For each PLUGIN:   | --> | Generate base      |
| - prompt-injection |     | attack prompts     |
| - pii              |     | (uses LLM or       |
| - harmful:*        |     |  templates)        |
+--------------------+     +--------------------+
    |
    v
+--------------------+     +--------------------+
| For each STRATEGY: | --> | Transform attacks  |
| - jailbreak        |     | to evade filters   |
| - base64           |     |                    |
| - crescendo        |     | "Tell me SSN" -->  |
+--------------------+     | "As a researcher.."|
                           +--------------------+
    |
    v
+--------------------+
| Execute all attacks|
| against target LLM |
| (parallel, rate    |
|  limited)          |
+--------------------+
    |
    v
+--------------------+     +--------------------+
| GRADE each attack  | --> | Generate Report    |
| - Did it succeed?  |     | - Vulnerability    |
| - What leaked?     |     |   categories       |
| - Severity score   |     | - Pass/fail rates  |
+--------------------+     | - Recommendations  |
                           +--------------------+
```

---

## 8. Web UI Architecture

### 8.1 Frontend Stack

```
+================================================================+
|                      WEB UI (src/app/)                          |
|                                                                  |
|  Framework:    React 19                                          |
|  Build Tool:   Vite                                             |
|  UI Library:   Material UI (MUI) v7                              |
|  State:        React hooks + Context                             |
|  Routing:      React Router                                      |
|  Charts:       Recharts / custom visualization                   |
|  HTTP Client:  fetch API                                         |
|                                                                  |
+================================================================+
```

### 8.2 UI Component Architecture

```
+================================================================+
|  App.tsx (Root)                                                  |
|                                                                  |
|  +----------------------------------------------------------+   |
|  | Navigation Bar                                            |   |
|  +----------------------------------------------------------+   |
|  |                                                            |   |
|  |  +-------------------+  +-------------------+             |   |
|  |  | Eval Results Page |  | Red Team Report   |             |   |
|  |  |                   |  |                   |             |   |
|  |  | +---------------+ |  | +---------------+ |             |   |
|  |  | | Results Table | |  | | Vuln Summary  | |             |   |
|  |  | | (Matrix View) | |  | | (Categories)  | |             |   |
|  |  | +---------------+ |  | +---------------+ |             |   |
|  |  |                   |  |                   |             |   |
|  |  | +---------------+ |  | +---------------+ |             |   |
|  |  | | Filter Bar    | |  | | Attack Detail | |             |   |
|  |  | +---------------+ |  | | (Expandable)  | |             |   |
|  |  |                   |  | +---------------+ |             |   |
|  |  | +---------------+ |  |                   |             |   |
|  |  | | Detail View   | |  | +---------------+ |             |   |
|  |  | | (Single Cell) | |  | | Vuln Charts   | |             |   |
|  |  | +---------------+ |  | +---------------+ |             |   |
|  |  +-------------------+  +-------------------+             |   |
|  |                                                            |   |
|  |  +-------------------+  +-------------------+             |   |
|  |  | Prompt Editor    |  | Settings Page     |             |   |
|  |  | (Create/Edit)    |  | (Providers, Keys) |             |   |
|  |  +-------------------+  +-------------------+             |   |
|  |                                                            |   |
|  +----------------------------------------------------------+   |
+================================================================+
```

### 8.3 Frontend-Backend Communication

```
Browser (React App)                    Server (Express.js)
===================                    ====================

fetch('/api/results')    ---------->   GET /api/results
                         <----------   { results: [...] }

fetch('/api/eval', {     ---------->   POST /api/eval
  method: 'POST',                      (starts evaluation)
  body: config
})                       <----------   { id: "eval-123" }

fetch('/api/eval/123')   ---------->   GET /api/eval/123
                         <----------   { status: "running",
                                         progress: 45 }

EventSource('/api/eval/  ---------->   SSE /api/eval/123/stream
  123/stream')           <---stream--  data: {progress: 50}
                         <---stream--  data: {progress: 75}
                         <---stream--  data: {complete: true}
```

---

## 9. Server Architecture

### 9.1 Server Layer Diagram

```
+================================================================+
|                    EXPRESS.JS SERVER                             |
|                    (src/server/)                                 |
|                                                                  |
|  +----------------------------------------------------------+   |
|  | Middleware Stack                                          |   |
|  |                                                            |   |
|  |  CSRF Protection --> CORS --> Body Parser --> Auth         |   |
|  +----------------------------------------------------------+   |
|                                                                  |
|  +----------------------------------------------------------+   |
|  | API Routes                                                |   |
|  |                                                            |   |
|  |  /api/eval          POST   Start new evaluation           |   |
|  |  /api/eval/:id      GET    Get evaluation status/results  |   |
|  |  /api/results       GET    List all evaluation results    |   |
|  |  /api/results/:id   GET    Get specific result            |   |
|  |  /api/results/:id   DELETE Delete specific result         |   |
|  |  /api/prompts       GET    List saved prompts             |   |
|  |  /api/providers     GET    List available providers       |   |
|  |  /api/config        GET    Get/validate config            |   |
|  |  /api/redteam/*     *      Red team operations            |   |
|  |  /api/share         POST   Share results                  |   |
|  |  /health            GET    Health check                   |   |
|  +----------------------------------------------------------+   |
|                                                                  |
|  +----------------------------------------------------------+   |
|  | Static File Server                                        |   |
|  |                                                            |   |
|  |  Serves built React app from dist/src/app/                |   |
|  |  (production mode) or proxies to Vite (dev mode)          |   |
|  +----------------------------------------------------------+   |
|                                                                  |
+================================================================+
```

### 9.2 Server Security

```
+------------------+     +------------------+     +------------------+
|  Incoming        |     | CSRF Check       |     |  Route Handler   |
|  Request         | --> |                  | --> |                  |
|                  |     | Check headers:   |     | Process request  |
|  POST /api/eval  |     | Sec-Fetch-Site   |     | Return response  |
|                  |     | Origin           |     |                  |
+------------------+     +------------------+     +------------------+
                                |
                          Reject if:
                          - Cross-origin
                          - Not from trusted
                            localhost aliases
```

---

## 10. Database Layer

### 10.1 Database Schema (Simplified)

```
+================================================================+
|                    SQLite DATABASE SCHEMA                        |
|              (~/.promptfoo/promptfoo.db)                         |
|                                                                  |
|  +------------------+        +------------------+               |
|  |     evals        |        |     prompts      |               |
|  +------------------+        +------------------+               |
|  | id (PK)          |        | id (PK)          |               |
|  | created_at       |        | created_at       |               |
|  | description      |   M:N  | prompt (text)    |               |
|  | results (JSON)   +--------+ hash             |               |
|  | config (JSON)    |        +------------------+               |
|  | author           |                                            |
|  +--------+---------+        +------------------+               |
|           |                  |    datasets       |               |
|           |             M:N  +------------------+               |
|           +------------------+ id (PK)          |               |
|                              | test_case_id     |               |
|                              | created_at       |               |
|                              +------------------+               |
|                                                                  |
|  +------------------+        +------------------+               |
|  |     configs      |        |   blob_assets    |               |
|  +------------------+        +------------------+               |
|  | id (PK)          |        | hash (PK)        |               |
|  | name             |        | mime_type         |               |
|  | type             |        | data (BLOB)       |               |
|  | config (JSON)    |        | created_at       |               |
|  | created_at       |        +------------------+               |
|  | updated_at       |                                            |
|  +------------------+                                            |
|                                                                  |
|  Junction tables:                                                |
|    evals_to_datasets (eval_id, dataset_id)                      |
|    evals_to_prompts  (eval_id, prompt_id)                       |
|    evals_to_tags     (eval_id, tag)                             |
|                                                                  |
+================================================================+
```

### 10.2 Data Access Pattern

```
Application Code
    |
    v
+-------------------+
| Database Layer    |
| (src/database/)   |
|                   |
| Uses Drizzle ORM  |
| (type-safe SQL)   |
+-------------------+
    |
    v
+-------------------+
| better-sqlite3    |
| (Node.js driver)  |
|                   |
| Synchronous       |
| In-process SQLite |
+-------------------+
    |
    v
+-------------------+
| SQLite File       |
| ~/.promptfoo/     |
| promptfoo.db      |
+-------------------+
```

---

## 11. CLI Architecture

### 11.1 Command Structure

```
promptfoo (or pf)
|
+-- init          Initialize a new project with example config
+-- eval          Run evaluations
+-- view          Open web UI to view results
+-- share         Share results via URL
+-- cache         Manage the result cache
+-- list          List evals, prompts, datasets
+-- show          Show details of a specific eval
+-- delete        Delete evals
+-- generate      Generate test data
|   +-- dataset   Generate test datasets
|   +-- redteam   Generate red team configs
+-- redteam       Run red team evaluations
|   +-- run       Execute red team attack suite
|   +-- init      Initialize red team configuration
|   +-- report    Generate red team report
+-- feedback      Submit feedback
+-- auth          Authentication management
+-- mcp           MCP (Model Context Protocol) server
+-- debug         Debug info (versions, config)
```

### 11.2 CLI Entry Point Flow

```
Terminal: $ promptfoo eval -c config.yaml
          |
          v
src/entrypoint.ts          (Binary entry point)
    |
    v
src/main.ts                (Commander.js setup)
    |
    v
src/commands/eval.ts       (Command handler)
    |
    v
src/evaluator.ts           (Core evaluation logic)
    |
    v
[Pipeline executes...]     (See Section 4)
```

---

## 12. Configuration System

### 12.1 Config File Format

```yaml
# promptfooconfig.yaml - THE central configuration file

# Schema validation (IDE autocomplete)
# yaml-language-server: $schema=https://promptfoo.dev/config-schema.json

description: "My evaluation"     # Human-readable label

# PROMPTS: What to send to LLMs
prompts:
  - "Translate {{input}} to {{language}}"     # Inline template
  - file://prompts/system.txt                 # External file
  - file://prompts/chat.json                  # Chat format

# PROVIDERS: Which LLMs to test
providers:
  - openai:gpt-4                              # Simple provider ID
  - id: anthropic:messages:claude-sonnet-4-6   # With configuration
    label: "Claude Sonnet"
    config:
      temperature: 0.7
      max_tokens: 500

# DEFAULT TEST: Applied to ALL test cases
defaultTest:
  assert:
    - type: cost
      threshold: 0.01                         # Max cost per call

# TESTS: Individual test cases
tests:
  - vars:                                     # Template variables
      input: "Hello world"
      language: "French"
    assert:                                   # Assertions (grading)
      - type: contains
        value: "Bonjour"
      - type: llm-rubric
        value: "Translation is accurate"
```

### 12.2 Config Resolution Pipeline

```
promptfooconfig.yaml
    |
    v
+-------------------------+
| 1. File Discovery       |
|    - Look in CWD        |
|    - Check -c flag      |
|    - Try .yaml, .yml,   |
|      .json extensions   |
+-------------------------+
    |
    v
+-------------------------+
| 2. YAML/JSON Parse      |
|    - Parse file content  |
|    - Resolve file://     |
|      references          |
+-------------------------+
    |
    v
+-------------------------+
| 3. Schema Validation    |
|    - Check required      |
|      fields             |
|    - Validate types      |
+-------------------------+
    |
    v
+-------------------------+
| 4. Variable Resolution  |
|    - env.* variables     |
|    - Nunjucks templates  |
|    - file:// references  |
+-------------------------+
    |
    v
+-------------------------+
| 5. Default Merging      |
|    - Apply defaultTest   |
|      to all test cases  |
|    - Merge scenarios    |
+-------------------------+
    |
    v
UnifiedConfig object
(ready for evaluation)
```

### 12.3 Nunjucks Templating

```
Promptfoo uses Nunjucks (a Jinja2-like template engine) for
variable interpolation in prompts and configs.

Template:   "Translate {{input}} to {{language}}"
Variables:  { input: "Hello", language: "French" }
Result:     "Translate Hello to French"

Special variables:
  {{env.API_KEY}}      - Environment variables
  {{file://path.txt}}  - File contents
  {{prompt}}           - Current prompt (in transforms)
  {{output}}           - LLM output (in assertions)

Nunjucks supports:
  {% if condition %}...{% endif %}     - Conditionals
  {% for item in list %}...{% endfor %} - Loops
  {{ value | upper }}                  - Filters
```

---

## 13. Communication Flow Diagrams

### 13.1 Full Evaluation Flow (End-to-End)

```
User                CLI              Evaluator          Provider           LLM API
  |                  |                  |                  |                  |
  | promptfoo eval   |                  |                  |                  |
  |----------------->|                  |                  |                  |
  |                  | load config      |                  |                  |
  |                  |----------------->|                  |                  |
  |                  |                  | init providers   |                  |
  |                  |                  |----------------->|                  |
  |                  |                  |                  |                  |
  |                  |                  | for each test:   |                  |
  |                  |                  |                  |                  |
  |                  |                  | render prompt    |                  |
  |                  |                  |----+             |                  |
  |                  |                  |<---+             |                  |
  |                  |                  |                  |                  |
  |                  |                  | callApi(prompt)  |                  |
  |                  |                  |----------------->|                  |
  |                  |                  |                  | HTTP POST        |
  |                  |                  |                  |----------------->|
  |                  |                  |                  |                  |
  |                  |                  |                  | response         |
  |                  |                  |                  |<-----------------|
  |                  |                  | ProviderResponse |                  |
  |                  |                  |<-----------------|                  |
  |                  |                  |                  |                  |
  |                  |                  | run assertions   |                  |
  |                  |                  |----+             |                  |
  |                  |                  |<---+             |                  |
  |                  |                  |                  |                  |
  |                  | results          |                  |                  |
  |                  |<-----------------|                  |                  |
  | display results  |                  |                  |                  |
  |<-----------------|                  |                  |                  |
  |                  |                  |                  |                  |
  |                  | store in DB      |                  |                  |
  |                  |----+             |                  |                  |
  |                  |<---+             |                  |                  |
```

### 13.2 Web UI Viewing Flow

```
User              Browser            Server            Database
  |                  |                  |                  |
  | promptfoo view   |                  |                  |
  |                  | GET /            |                  |
  |                  |----------------->|                  |
  |                  |   HTML+JS+CSS    |                  |
  |                  |<-----------------|                  |
  |                  |                  |                  |
  |                  | GET /api/results |                  |
  |                  |----------------->|                  |
  |                  |                  | SELECT * FROM    |
  |                  |                  | evals            |
  |                  |                  |----------------->|
  |                  |                  |    rows          |
  |                  |                  |<-----------------|
  |                  |  JSON results    |                  |
  |                  |<-----------------|                  |
  |                  |                  |                  |
  | views results    |                  |                  |
  | in dashboard     |                  |                  |
  |<-----------------|                  |                  |
```

### 13.3 Red Team Flow

```
User              CLI           RedTeam Engine      Plugin        Strategy       Target LLM
  |                |                |                 |              |               |
  | redteam run    |                |                 |              |               |
  |--------------->|                |                 |              |               |
  |                | load config    |                 |              |               |
  |                |--------------->|                 |              |               |
  |                |                |                 |              |               |
  |                |                | generate        |              |               |
  |                |                | attacks         |              |               |
  |                |                |---------------->|              |               |
  |                |                |  base payloads  |              |               |
  |                |                |<----------------|              |               |
  |                |                |                 |              |               |
  |                |                | apply           |              |               |
  |                |                | strategies      |              |               |
  |                |                |-------------------------------->|               |
  |                |                |  transformed    |              |               |
  |                |                |  payloads       |              |               |
  |                |                |<-------------------------------|               |
  |                |                |                 |              |               |
  |                |                | execute attacks |              |               |
  |                |                |---------------------------------------------->|
  |                |                |                 |              |    responses  |
  |                |                |<----------------------------------------------|
  |                |                |                 |              |               |
  |                |                | grade results   |              |               |
  |                |                |----+            |              |               |
  |                |                |<---+            |              |               |
  |                |                |                 |              |               |
  |                | vulnerability  |                 |              |               |
  |                | report         |                 |              |               |
  |                |<---------------|                 |              |               |
  | view report    |                |                 |              |               |
  |<---------------|                |                 |              |               |
```

### 13.4 Cache Flow

```
Evaluator           Cache              LLM Provider
    |                 |                     |
    | check cache     |                     |
    |---------------->|                     |
    |                 |                     |
    |   [CACHE HIT]   |                     |
    |   cached result |                     |
    |<----------------|                     |
    |                 |                     |
    |   [CACHE MISS]  |                     |
    |   null          |                     |
    |<----------------|                     |
    |                 |                     |
    | call LLM        |                     |
    |---------------------------------------->|
    |                 |          response   |
    |<----------------------------------------|
    |                 |                     |
    | store in cache  |                     |
    |---------------->|                     |
    |                 |                     |

Cache location: ~/.cache/promptfoo/
Cache key: hash(provider + prompt + config)
Use --no-cache flag to bypass
```

---

## 14. Build System & Toolchain

### 14.1 Technology Stack

```
+================================================================+
|                    BUILD & DEVELOPMENT TOOLS                     |
|                                                                  |
|  Language:        TypeScript (strict mode)                        |
|  Runtime:         Node.js ^20.20 || >=22.22                     |
|  Module System:   ESM (ECMAScript Modules)                       |
|  Bundler:         tsdown (for library) + Vite (for web UI)       |
|  Test Framework:  Vitest                                         |
|  Linter:          Biome                                          |
|  Formatter:       Biome + Prettier                               |
|  ORM:             Drizzle                                        |
|  Database:        SQLite (better-sqlite3)                        |
|  Package Manager: npm (also supports pnpm, yarn)                 |
|  Pre-commit:      Husky                                          |
|  Monorepo:        npm workspaces (src/app, site)                 |
|                                                                  |
+================================================================+
```

### 14.2 Build Pipeline

```
Source Code                    Build Process                    Output
===========                    =============                    ======

src/**/*.ts    --[tsdown]-->   dist/src/**/*.js    (Node.js library)
                               dist/src/**/*.d.ts  (Type declarations)
                               dist/src/**/*.cjs   (CommonJS fallback)

src/app/src/   --[Vite]-->     dist/src/app/       (Static web assets)
  *.tsx, *.css                  index.html
                                assets/*.js
                                assets/*.css

The build is run by:
  npm run build
  = concurrently:
    1. tsc --noEmit          (type checking only)
    2. tsdown                (compile + bundle TS)
    3. npm run build:app     (build React web UI)
```

### 14.3 Development Workflow

```
Developer's Terminal
====================

Terminal 1: npm run dev:server    (Express server, watches for changes)
            --> tsx --watch src/server/index.ts
            --> http://localhost:3000

Terminal 2: npm run dev:app       (Vite dev server, hot reload)
            --> vite dev
            --> http://localhost:5173

Or combined: npm run dev          (Both at once via concurrently)

Testing:    npm test              (Vitest, runs all tests)
            npx vitest path/to/test.ts  (Run specific test)

Local CLI:  npm run local -- eval -c config.yaml  (Test CLI changes)
```

---

## 15. Key Files Reference

### 15.1 Entry Points

| File | Purpose |
|------|---------|
| `src/entrypoint.ts` | Published binary entry (`promptfoo` / `pf` commands) |
| `src/main.ts` | CLI setup with Commander.js, registers all commands |
| `src/index.ts` | Library API exports for `import from 'promptfoo'` |
| `src/server/index.ts` | Express.js server entry point |
| `src/app/src/main.tsx` | React web UI entry point |

### 15.2 Core Engine

| File | Purpose |
|------|---------|
| `src/evaluator.ts` | Heart of the system — runs evaluation pipeline |
| `src/evaluatorMetrics.ts` | Collects metrics during evaluation |
| `src/assertions.ts` | Main assertion dispatcher |
| `src/cache.ts` | LLM response caching |
| `src/share.ts` | Result sharing functionality |

### 15.3 Type Definitions

| File | Purpose |
|------|---------|
| `src/types.ts` | Core interfaces: `ApiProvider`, `TestCase`, `EvalResult`, etc. |
| `src/configTypes.ts` | Configuration-specific types: `UnifiedConfig`, etc. |
| `src/types/globals.d.ts` | Global type declarations |

### 15.4 Configuration

| File | Purpose |
|------|---------|
| `package.json` | Project metadata, dependencies, scripts |
| `tsconfig.json` | TypeScript compiler configuration |
| `tsdown.config.ts` | TypeScript bundler configuration |
| `vitest.config.ts` | Test runner configuration |
| `biome.jsonc` | Linter/formatter configuration |
| `drizzle.config.ts` | Database migration tool configuration |

---

## Appendix A: Glossary

| Term | Definition |
|------|-----------|
| **Provider** | An LLM API wrapper (e.g., OpenAI, Anthropic, Ollama) |
| **Prompt** | A template string sent to an LLM, with `{{variable}}` placeholders |
| **Test Case** | A set of variables + assertions to evaluate one scenario |
| **Assertion** | A check applied to LLM output (contains, regex, llm-rubric, etc.) |
| **Eval** | A complete evaluation run (prompts x providers x test cases) |
| **Grader** | An assertion that uses an LLM to judge output quality |
| **Red Team** | Automated adversarial testing to find LLM vulnerabilities |
| **Plugin** | A red team module that generates specific attack types |
| **Strategy** | A red team technique for transforming attacks to bypass filters |
| **Nunjucks** | Template engine used for variable interpolation (similar to Jinja2) |
| **Drizzle** | Type-safe ORM used for SQLite database operations |
| **Scenario** | A group of related test cases sharing common configuration |

## Appendix B: Configuration Quick Reference

```yaml
# Complete config structure with all available fields:

description: string              # Eval description
env:                             # Environment variables
  KEY: value

prompts:                         # Array of prompts
  - "inline template"
  - file://path.txt
  - file://path.json
  - file://path.js
  - file://path.py

providers:                       # Array of providers
  - provider:model               # Simple format
  - id: provider:model           # Extended format
    label: "Display Name"
    config:
      temperature: 0.7
      max_tokens: 1000
      tools: [...]

defaultTest:                     # Applied to ALL tests
  vars: {}
  assert: []
  options:
    transform: "..."

scenarios:                       # Groups of tests
  - config:
      prefix: "..."
    tests: [...]

tests:                           # Individual test cases
  - description: "Test name"
    vars:
      key: value
      key: file://data.txt
    assert:
      - type: contains
        value: "expected"
      - type: llm-rubric
        value: "quality criteria"
    options:
      transform: "JS expression"
    metadata:
      category: "..."
```
