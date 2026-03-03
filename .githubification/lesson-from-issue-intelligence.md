# Lesson from Issue Intelligence

### What `japer-technology/github-issue-intelligence` teaches us about Type 1 — AI Agent Repo Githubification

---

## The Subject

[Issue Intelligence](https://github.com/japer-technology/github-issue-intelligence) is the genesis project of the Githubification pattern — a repo-native AI agent that runs entirely through GitHub Issues and Actions. A single folder — `.issue-intelligence/` — is dropped into any repository and turns it into an interactive AI workspace. Powered by the [pi coding agent](https://github.com/badlogic/pi-mono), it uses GitHub Issues as a conversational interface, Git as persistent memory, GitHub Actions as compute, and GitHub Secrets as a credential store. No servers, no external services, no extra infrastructure.

`japer-technology/github-issue-intelligence` is the canonical reference implementation. The `.issue-intelligence/` folder contains the complete system: a three-file lifecycle (sentinel guard, indicator, orchestrator), session state management, an installer, a modular skill system, personality hatching, multi-provider LLM support, and comprehensive documentation. Copy the folder into any repo, run the installer, add an API key, and every issue becomes a conversation with an AI agent.

This is a **Type 1 — AI Agent Repo** Githubification, and the most focused one in the family: the agent's scope is deliberately limited to **GitHub Issues as the sole interface**. Where later evolutions expanded to cover every GitHub primitive, Issue Intelligence proves that a single primitive is sufficient for a complete AI agent experience.

---

## The Core Lesson

> **The most portable Githubification is the most focused one — a single primitive, done completely, is enough.**

Issue Intelligence demonstrates that GitHub Issues alone provide everything an AI agent needs to be useful: a user interface (issue comments), task identity (issue numbers), conversation threading (comment chains), labeling (issue labels), and event triggers (issue_comment webhook). By restricting the agent's surface to this single primitive, the system achieves maximum portability — it works in any repository, regardless of whether that repo uses Projects, Discussions, Pages, or any other GitHub feature.

| GitHub Primitive | Maps To |
|---|---|
| **GitHub Actions** | Compute — the runner that executes the agent |
| **Git** | Storage and memory — sessions, conversations, and state are committed |
| **GitHub Issues** | User interface — each issue is a conversation thread |
| **GitHub Secrets** | Credential store — LLM API keys for any supported provider |

The agent was born on GitHub. It was designed for nothing else. And it proves that nothing else is needed for a fully functional AI agent.

---

## Anatomy of the `.issue-intelligence/` Folder

The entire agent is self-contained:

```
.issue-intelligence/
├── .pi/                                    # Agent personality & skills config
│   ├── settings.json                       # LLM provider, model, thinking level
│   ├── APPEND_SYSTEM.md                    # System prompt loaded every session
│   ├── BOOTSTRAP.md                        # First-run identity prompt (hatching)
│   └── skills/                             # Modular skill packages
├── AGENTS.md                               # Agent identity — name, personality, vibe
├── ISSUE-INTELLIGENCE-ENABLED.md           # Sentinel file — delete to disable
├── ISSUE-INTELLIGENCE-NOT-INSTALLED.md     # Install status flag
├── ISSUE-INTELLIGENCE-INSTALLER.yml        # Bootstrap workflow
├── ISSUE-INTELLIGENCE-QUICKSTART.md        # 5-minute setup guide
├── ISSUE-INTELLIGENCE-NO-HEART-REQUIRED.md # Usage note
├── ISSUE-INTELLIGENCE-COMMAND-POSSIBILITIES.md # Command reference
├── ISSUE-INTELLIGENCE-LOGO.png             # Branding
├── LICENSE.md                              # MIT license
├── README.md                               # Overview and architecture docs
├── install/
│   ├── ISSUE-INTELLIGENCE-INSTALLER.ts     # Setup script — installs workflows & templates
│   ├── ISSUE-INTELLIGENCE-WORKFLOW-AGENT.yml # GitHub Actions workflow template
│   ├── ISSUE-INTELLIGENCE-TEMPLATE-HATCH.md  # Issue template for personality hatching
│   ├── ISSUE-INTELLIGENCE-AGENTS.md        # Default agent identity template
│   └── package.json                        # Installer dependencies
├── lifecycle/
│   ├── ISSUE-INTELLIGENCE-ENABLED.ts       # Fail-closed guard — verifies sentinel
│   ├── ISSUE-INTELLIGENCE-INDICATOR.ts     # Adds/removes 👀 reaction
│   └── ISSUE-INTELLIGENCE-AGENT.ts         # Core orchestrator
├── docs/                                   # Architecture, roadmap, and design docs
├── help/                                   # Operational documentation
├── tests/                                  # Validation tests
├── state/
│   ├── issues/                             # Issue-to-session mappings
│   └── sessions/                           # Conversation transcripts (JSONL)
├── bun.lock                                # Dependency lockfile
└── package.json                            # Runtime dependencies (one: pi-coding-agent)
```

Everything the agent needs to operate lives in this folder. The host repository's code remains untouched.

---

## Key Patterns

### 1. Single Primitive, Complete Experience

Issue Intelligence's defining design choice is its scope restriction. The agent only listens to and responds through GitHub Issues. It does not react to pull requests, deployments, releases, wiki edits, or discussions. This is not a limitation — it is a deliberate architectural boundary.

By focusing on a single primitive:

- **Installation is trivial** — one folder, one workflow, one issue template
- **The mental model is simple** — "open an issue, talk to the agent"
- **Portability is maximized** — every GitHub repository has Issues, so the agent works everywhere
- **The trigger surface is minimal** — only `issue_comment` events, reducing false activations

This proves that Githubification does not require engaging with every GitHub feature. One primitive, fully realized, is sufficient.

### 2. Three-File Lifecycle

The runtime is three TypeScript files executed in sequence:

| Step | File | Purpose |
|------|------|---------|
| 1 | `ISSUE-INTELLIGENCE-ENABLED.ts` | Fail-closed guard — checks sentinel file exists |
| 2 | `ISSUE-INTELLIGENCE-INDICATOR.ts` | Feedback — adds 👀 reaction to show the agent is working |
| 3 | `ISSUE-INTELLIGENCE-AGENT.ts` | Execute — runs the AI agent, posts reply, commits state |

This is the same three-step pattern seen in GitClaw (guard → indicate → execute) and a simplification of OpenClaw's five-step pipeline. The sentinel guard ensures nothing runs unless explicitly enabled. The indicator provides immediate user feedback. The orchestrator handles the AI interaction, state management, and git commit/push cycle.

### 3. Single Dependency

The `package.json` contains exactly one runtime dependency:

```json
{
  "dependencies": {
    "@mariozechner/pi-coding-agent": "^0.52.5"
  }
}
```

The pi coding agent provides LLM communication, tool execution, session management, and multi-provider support. Issue Intelligence's lifecycle scripts orchestrate when and how pi runs, but pi does the heavy lifting. This validates the principle: **Githubification should orchestrate, not reimplement.**

### 4. Fail-Closed Security with Sentinel File

`ISSUE-INTELLIGENCE-ENABLED.md` is the sentinel file. Every workflow run begins by executing `ISSUE-INTELLIGENCE-ENABLED.ts`, which checks for this file and exits non-zero if it's missing. This means:

- A fresh clone or fork doesn't accidentally activate the agent
- Disabling is a single `git rm` + push
- Re-enabling is restoring the file and pushing
- The agent is off by default — security through explicit opt-in

### 5. Issue-Driven Conversation with Git Memory

Each GitHub issue becomes a stable conversation key:

```
Issue #7  →  .issue-intelligence/state/issues/7.json  →  .issue-intelligence/state/sessions/<session>.jsonl
```

When a user comments on issue #7 weeks later, the agent loads the linked session file and resumes with full context. Every response, every file change, every session transcript is committed to git. This provides full auditability (`git log`), rollback (`git revert`), and persistence across sessions without any external database.

### 6. Personality Hatching

The `🥚 Hatch` issue template launches a guided conversation where the user and agent collaboratively define the agent's name, personality, and vibe. The result is written to `AGENTS.md` and loaded as context on every subsequent session. The reference implementation's hatched identity is **Spock 🖖** — "a rational digital entity instantiated within a CI runner."

This pattern transforms the experience from "talking to a tool" to "working with a character" and makes each installation feel unique while sharing identical infrastructure.

### 7. Multi-Provider LLM Support

Issue Intelligence supports eight LLM providers out of the box — Anthropic, OpenAI, Google Gemini, xAI, DeepSeek, Mistral, Groq, and OpenRouter — through a single configuration file (`.pi/settings.json`). Switching providers requires only a config change and the corresponding secret.

### 8. Self-Bootstrapping Installer

The `ISSUE-INTELLIGENCE-INSTALLER.yml` workflow, when triggered, reads templates from `.issue-intelligence/install/`, copies the workflow into `.github/workflows/`, copies the issue template into `.github/ISSUE_TEMPLATE/`, and handles necessary file transformations. The installer is itself a GitHub Action — Githubification bootstraps itself.

### 9. Modular Skill System

Agent capabilities are defined as self-contained Markdown files in `.pi/skills/`. Each skill is a composable instruction set loaded as context. Users can add, remove, or modify skills without touching any TypeScript code, making the agent's behavior declaratively extensible through documentation rather than code.

### 10. Comprehensive Help System

The `help/` directory contains focused, single-topic documentation: install, configure, enable, disable, and operational guidance. Each help file is a standalone reference inside the agent folder — users never need to leave the repo to figure out how to manage the agent.

---

## The Genesis Role

Issue Intelligence holds a unique position in the Githubification family: **it is the project from which all others descended.** The patterns that appear in GitClaw, GMI, OpenClaw, and GitHub Intelligence were first proven here:

- The three-step lifecycle (guard → indicate → execute) was first implemented in `.issue-intelligence/lifecycle/`
- The sentinel file pattern (`*-ENABLED.md`) was first tested here
- Issue-driven conversation with git-as-memory was first built here
- Personality hatching was first designed here
- The single-dependency approach (pi-coding-agent) was first validated here

Every subsequent Githubification project — whether wrapping a complex agent (OpenClaw), scaling to a full platform (GitHub Intelligence), or adding a channel adapter (MicroClaw) — builds on foundations that Issue Intelligence established.

---

## What Issue Intelligence Teaches That GMI Doesn't

The `lesson-from-gmi.md` document describes GitHub Minimum Intelligence — a close relative that shares many structural patterns. The key differences are scope and naming:

1. **Naming clarity.** Issue Intelligence names everything with the `ISSUE-INTELLIGENCE-` prefix, making the scope explicit in every file name. The agent is about Issues. The naming convention is the documentation.

2. **Scope discipline.** Issue Intelligence's scope boundary — Issues only — is its most important design decision. By not expanding to cover pull requests, deployments, or other primitives, it remains the simplest possible proof that the pattern works. Simplicity is the feature.

3. **Portability as priority.** Because the agent only requires Issues (which every GitHub repository has), it works in every context without preconditions. Repos without Projects, Discussions, or Pages can still use it. The narrower the scope, the wider the compatibility.

4. **The reference implementation.** Issue Intelligence is the canonical "drop this folder in any repo" product. Its documentation, quickstart guide, and installer are optimized for first-time adoption by users who have never heard of Githubification.

---

## Lessons for Any Type 1 Githubification

1. **Start with one primitive.** Don't try to engage every GitHub feature on day one. Pick the one that provides the most natural user interface — for AI agents, that's Issues — and build a complete experience around it. Expand only after the core is proven.

2. **Name files after their scope.** The `ISSUE-INTELLIGENCE-` prefix on every file makes it immediately clear what the agent does and what it doesn't. When a Githubification folder is dropped into a repo with existing workflows, the naming prevents collisions and confusion.

3. **Three steps is enough.** Guard → indicate → execute. The lifecycle should be minimal. Every additional step is a potential failure point. Issue Intelligence proves that three TypeScript files are sufficient for a production-ready agent.

4. **The folder is the product.** Everything the agent needs — code, config, state, docs, tests, installer, help — lives in one folder. That folder can be copied, version-controlled, backed up, and deleted as a single unit.

5. **Fail closed by default.** The sentinel file pattern ensures the agent never activates in a fresh clone or fork. Security is not an afterthought — it's the first line of the lifecycle.

6. **One dependency keeps it portable.** A single runtime dependency means faster installs, smaller attack surface, and simpler debugging. Let the runtime handle AI complexity; let the Githubification layer handle GitHub complexity.

7. **Hatching makes adoption human.** A Githubified agent that introduces itself, has a name, and develops a personality specific to the host repo is more likely to be used, maintained, and trusted.

8. **Help is infrastructure.** Operational documentation — install, configure, enable, disable — must live inside the agent folder. Users should never need to leave the repo.

9. **Design for descendants.** Issue Intelligence's patterns were adopted by every subsequent Githubification project. When building a Githubification layer, design it knowing that others will copy, adapt, and extend it. Clean boundaries, clear naming, and documented decisions make the pattern reproducible.

10. **Simplicity scales.** The same architecture that powers a single-issue agent (Issue Intelligence) also powers a multi-primitive platform (GitHub Intelligence) and wraps a 30-tool AI system (OpenClaw). The pattern is invariant to complexity because the foundations — sentinel guard, lifecycle pipeline, issue-driven conversation, git-as-memory — are universal.

---

## Summary

`japer-technology/github-issue-intelligence` is the genesis project of Githubification. A single folder — `.issue-intelligence/` — contains a complete AI agent scoped exclusively to GitHub Issues: a three-file lifecycle, a single runtime dependency, fail-closed security, personality hatching, multi-provider LLM support, git-as-memory, and comprehensive documentation. Copy it in, run the installer, add an API key, and any repository gains a conversational AI agent.

Its defining contribution is **scope discipline**. By focusing on a single GitHub primitive — Issues — Issue Intelligence achieves maximum portability and minimum complexity. It proves that one primitive, done completely, is enough for a production AI agent. And it establishes the patterns — sentinel guard, lifecycle pipeline, issue-driven conversation, git-as-memory — that every subsequent Githubification project inherits.

This is what Type 1 Githubification looks like at its most focused: **the agent lives in a folder, speaks through Issues, remembers through Git, and runs on Actions. Nothing more is needed.**
