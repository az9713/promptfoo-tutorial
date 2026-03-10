# Promptfoo Deep Dive Documentation

> **This is a companion documentation repo for [promptfoo](https://github.com/promptfoo/promptfoo) — the open-source LLM evaluation and red-teaming framework.**
>
> For the source code, official docs, and latest releases, visit the **original repo: [github.com/promptfoo/promptfoo](https://github.com/promptfoo/promptfoo)**
>
> Official website: [promptfoo.dev](https://www.promptfoo.dev)

---

## Why This Repo Exists

The official promptfoo project is a large, sophisticated codebase. This repo provides **comprehensive, beginner-friendly documentation** that covers the architecture, internals, and usage of promptfoo in depth — designed for developers and users who want to truly understand how everything works under the hood, not just follow quick-start recipes.

Whether you come from a C/C++/Java background and are new to full-stack web development, or you're a user who wants to go beyond the basics, these guides will take you from zero to hero.

## Recommended Video

For an excellent introduction to the problems promptfoo solves, watch this video by 1littlecoder:

**["OpenAI Just Solved AI's Biggest Security Problem"](https://www.youtube.com/watch?v=E8UObj_R2mM)**

It covers why LLM security testing matters and how promptfoo fits into the picture.

---

## Documentation Index

| Document | Audience | Description |
|----------|----------|-------------|
| [Architecture Guide](docs/ARCHITECTURE.md) | Developers | System design with ASCII diagrams at multiple abstraction levels, evaluation pipeline, provider/assertion/red-team systems, database schema, communication flows, key file reference |
| [Developer Guide](docs/DEVELOPER_GUIDE.md) | New Developers | Step-by-step environment setup, tech stack explained for C/Java developers, project structure walkthrough, how to read the code, making your first change, testing, debugging, git workflow |
| [User Quick Start](docs/USER_QUICKSTART.md) | Users | 11 hands-on use cases with complete configs: translation eval, model comparison, JSON validation, LLM-as-judge, RAG evaluation, moderation, multi-turn conversations, local models (Ollama), red teaming, CI/CD, custom assertions |
| [Study Plan](docs/STUDY_PLAN.md) | Learners | 11-phase zero-to-hero learning path with theory + implementation + hands-on exercises + checkpoints for each module, recommended file reading order, capstone projects |
| [Security Guide](docs/SECURITY_GUIDE.md) | Security Engineers | LLM threat landscape, how promptfoo addresses each threat, red team architecture, attack plugins and strategies, grading system, promptfoo's own security model, real-world scenarios |

---

## What is Promptfoo?

Promptfoo is a CLI and library for evaluating and red-teaming LLM applications. It lets you:

- **Test prompts and models** with automated evaluations and 70+ assertion types
- **Compare models side-by-side** (OpenAI, Anthropic, Google, Azure, Ollama, and 50+ more)
- **Red team your AI** with automated adversarial attacks to find security vulnerabilities
- **Automate checks in CI/CD** to catch regressions before deployment
- **Scan code** for LLM-related security issues in pull requests

## Quick Links to the Official Project

- [Original Repo (source code)](https://github.com/promptfoo/promptfoo)
- [Official Documentation](https://www.promptfoo.dev/docs/)
- [Getting Started Guide](https://www.promptfoo.dev/docs/getting-started/)
- [Red Teaming Guide](https://www.promptfoo.dev/docs/red-team/)
- [Supported Providers](https://www.promptfoo.dev/docs/providers/)
- [Discord Community](https://discord.gg/promptfoo)

---

*These docs were created by studying the promptfoo codebase (v0.121.1). For the latest changes, always check the [original repo](https://github.com/promptfoo/promptfoo).*
