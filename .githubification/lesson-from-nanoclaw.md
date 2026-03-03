# Lesson from NanoClaw

### What `japer-technology/github-nanoclaw` teaches us about Type 1 — AI Agent Repo Githubification

---

## The Subject

[NanoClaw](https://github.com/japer-technology/github-nanoclaw) is a personal AI assistant built on the Claude Agent SDK. It runs as a single Node.js process on macOS or Linux, routes messages from multiple channels (WhatsApp, Telegram, Discord, Slack, Gmail) to Claude agents executing inside isolated Linux containers, and maintains per-group memory and scheduled tasks backed by SQLite.

NanoClaw was created as a direct reaction to [OpenClaw](https://github.com/openclaw/openclaw). Where OpenClaw has nearly half a million lines of code, 53 config files, and 70+ dependencies with application-level security, NanoClaw provides the same core functionality in a codebase small enough for a single person — or a single AI — to read and understand: one process, a handful of source files, six runtime dependencies, and OS-level container isolation.

`japer-technology/github-nanoclaw` is a Type 1 — AI Agent Repo. The repository contains a fully functional AI agent designed for local execution via `launchd` (macOS) or `systemd` (Linux). It is not yet Githubified. What makes it instructive is not what it has done with GitHub Actions, but what its architecture reveals about how Githubification **should** work — and what a Githubification layer would look like for an agent whose entire design philosophy is anti-complexity.

---

## The Core Lesson

> **The best agent to Githubify is one that was already designed to be understood by an AI.**

NanoClaw's defining constraint is that the entire codebase must fit within an AI's context window. The README advertises "34.9k tokens, 17% of context window." Every design decision — single process, no microservices, no configuration sprawl, skills instead of features — follows from this constraint. The codebase is AI-legible by construction.

This matters for Githubification because the Githubification layer itself must understand the agent it wraps. When the agent is 500,000 lines of code with dozens of configuration files, the wrapping layer becomes complex — it must accommodate runtime variations, handle dependency matrices, and work around assumptions the agent makes about its environment. When the agent is 35,000 tokens with six dependencies, the wrapping layer can be minimal. The impedance mismatch between "what the agent expects" and "what GitHub provides" shrinks in proportion to the agent's complexity.

| GitHub Primitive | NanoClaw Equivalent |
|---|---|
| **GitHub Actions** | Compute — replaces the local Node.js process and launchd/systemd |
| **Git** | Storage and memory — replaces SQLite for messages, sessions, and task state |
| **GitHub Issues** | User interface — replaces WhatsApp/Telegram/Discord/Slack/Gmail channels |
| **GitHub Secrets** | Credential store — replaces `.env` files and local credential storage |

The mapping is clean because NanoClaw's architecture is clean. A single process with a polling loop, a SQLite database, and container invocations maps directly to a GitHub Actions workflow with git state and container steps.

---

## Anatomy of the Repository

NanoClaw is not organized into a single self-contained folder (unlike `.github-claw/` or `.GITOPENCLAW/`). It is a full standalone repository:

```
github-nanoclaw/
├── .claude/
│   └── skills/                        # Claude Code skill definitions
│       ├── setup/                     # /setup — first-time installation
│       ├── customize/                 # /customize — guided modifications
│       ├── debug/                     # /debug — troubleshooting
│       ├── update-nanoclaw/           # /update-nanoclaw — upstream merge
│       ├── add-whatsapp/              # /add-whatsapp — channel skill
│       ├── add-telegram/              # /add-telegram — channel skill
│       ├── add-discord/               # /add-discord — channel skill
│       ├── add-slack/                 # /add-slack — channel skill
│       ├── add-gmail/                 # /add-gmail — channel skill
│       ├── add-telegram-swarm/        # /add-telegram-swarm — agent teams
│       ├── add-voice-transcription/   # Voice message support
│       ├── add-parallel/              # Parallel agent execution
│       ├── convert-to-apple-container/# Switch Docker → Apple Container
│       ├── x-integration/             # X/Twitter integration
│       ├── qodo-pr-resolver/          # PR review integration
│       └── get-qodo-rules/            # Coding rules loader
├── src/
│   ├── index.ts                       # Orchestrator: state, message loop, agent invocation
│   ├── channels/
│   │   └── registry.ts                # Channel self-registration at startup
│   ├── container-runner.ts            # Spawns isolated agent containers
│   ├── container-runtime.ts           # Docker / Apple Container abstraction
│   ├── mount-security.ts             # Mount validation and path blocking
│   ├── ipc.ts                         # IPC watcher and task processing
│   ├── router.ts                      # Message formatting and outbound routing
│   ├── group-queue.ts                 # Per-group queue with concurrency limit
│   ├── group-folder.ts               # Group directory management
│   ├── task-scheduler.ts              # Scheduled task execution
│   ├── db.ts                          # SQLite operations
│   ├── config.ts                      # Trigger pattern, paths, intervals
│   ├── types.ts                       # Shared type definitions
│   ├── env.ts                         # Environment variable handling
│   └── logger.ts                      # Logging
├── container/
│   ├── Dockerfile                     # Agent container image (Node 22, Chromium, Claude Code)
│   ├── build.sh                       # Container build script
│   ├── agent-runner/                  # Code that runs inside the container
│   └── skills/                        # Container-level skills (e.g., browser automation)
├── skills-engine/                     # Programmatic skill apply/remove/update engine
│   ├── apply.ts                       # Three-way merge skill application
│   ├── uninstall.ts                   # Skill removal via replay
│   ├── rebase.ts                      # Flatten accumulated layers
│   ├── replay.ts                      # Deterministic skill reproduction
│   ├── manifest.ts                    # Skill manifest parsing
│   ├── state.ts                       # State tracking (.nanoclaw/state.yaml)
│   ├── backup.ts                      # Pre-operation backup/restore
│   └── ...                            # Additional engine modules
├── groups/
│   ├── main/                          # Main group (admin/self-chat)
│   └── global/                        # Global shared memory
├── setup/                             # Node.js setup modules
├── scripts/                           # Utility scripts (apply-skill, fix-drift, etc.)
├── docs/
│   ├── SPEC.md                        # Full architecture specification
│   ├── SECURITY.md                    # Security model documentation
│   ├── REQUIREMENTS.md                # Design decisions and philosophy
│   ├── SDK_DEEP_DIVE.md              # Claude Agent SDK analysis
│   └── ...                            # Additional architecture docs
├── CLAUDE.md                          # Agent context for Claude Code
├── CONTRIBUTING.md                    # Contribution guidelines (skills, not features)
├── setup.sh                           # Bootstrap script
├── package.json                       # 6 runtime dependencies
├── tsconfig.json                      # TypeScript config
└── vitest.config.ts                   # Test configuration
```

The architecture is a pipeline: channels receive messages → orchestrator polls and routes → container runner spawns an isolated Claude agent → agent responds → response routes back to the channel. State lives in SQLite. Memory lives in `groups/*/CLAUDE.md` files. Scheduled tasks run on a cron loop.

---

## Key Patterns

### 1. Anti-Complexity as Design Principle

NanoClaw's `docs/REQUIREMENTS.md` opens with a blunt assessment of OpenClaw:

> "That project became a monstrosity — 4-5 different processes running different gateways, endless configuration files, endless integrations. It's a security nightmare where agents don't run in isolated processes."

Every design choice follows from rejecting that complexity:

- **One process**, not microservices
- **Six runtime dependencies**, not seventy
- **SQLite**, not Postgres or Redis
- **Code changes**, not configuration files
- **Skills**, not features

For Githubification, this matters because **complexity is the enemy of wrapping**. The `.GITOPENCLAW/` layer in `github-openclaw` must accommodate 30+ tools, vector databases, browser automation, and sub-agent orchestration. A Githubification layer for NanoClaw needs to accommodate a single process that polls for messages and spawns containers. The wrapping surface area scales with the agent's surface area.

### 2. Container Isolation as Security Foundation

NanoClaw's security model is architectural, not application-level. Agents execute inside Linux containers (Docker or Apple Container) with:

- **Process isolation** — container processes cannot affect the host
- **Filesystem isolation** — only explicitly mounted directories are visible
- **Non-root execution** — runs as unprivileged `node` user (uid 1000)
- **Ephemeral containers** — fresh environment per invocation (`--rm`)
- **Mount security** — an external allowlist blocks `.ssh`, `.gnupg`, `.aws`, credentials, private keys
- **Credential filtering** — only `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY` are exposed

This is a higher security bar than application-level permission checks. For Githubification, this pattern maps naturally to GitHub Actions runners, which provide similar isolation: ephemeral VMs, restricted filesystem, secret injection via `${{ secrets.* }}`. The container-in-a-container model (NanoClaw container running inside a GitHub Actions runner) would provide defense in depth.

### 3. Skills Over Features

NanoClaw's contribution model is radical: **don't add features to the codebase; add skills that transform the codebase.**

A [Claude Code skill](https://code.claude.com/docs/en/skills) is a Markdown file in `.claude/skills/` that teaches Claude Code how to modify a NanoClaw installation. When a user runs `/add-telegram`, Claude Code reads the skill file and programmatically adds Telegram support to that user's fork — writing new channel handlers, updating the router, adding dependencies, configuring the container.

The `skills-engine/` directory implements this at scale: three-way merge via `git merge-file`, shared resolution caches via `git rerere`, deterministic state tracking in `.nanoclaw/state.yaml`, backup/restore safety, and a three-level conflict resolution model (git → Claude Code → Claude Code + user input).

This is the contribution model Githubification needs. A Githubified NanoClaw wouldn't need a separate skill for "run on GitHub Actions" if the Githubification layer itself were a skill: `/githubify` — a skill that teaches Claude Code how to add GitHub Actions workflows, convert SQLite state to git-committed JSON, replace channel polling with issue event triggers, and wire up secrets. The user's fork gets clean, working Githubification code that does exactly what they need.

### 4. AI-Native Development

NanoClaw assumes Claude Code is always present:

- **No installation wizard** — Claude Code guides setup via `/setup`
- **No monitoring dashboard** — ask Claude what's happening
- **No debugging tools** — describe the problem and Claude fixes it
- **No configuration files** — tell Claude Code what you want changed

The `CLAUDE.md` at the repo root provides structured context: key files, skill definitions, development commands, troubleshooting tips. This file is the agent's self-knowledge — it tells Claude Code how to operate on the codebase.

For Githubification, this is significant because it means the Githubification layer can rely on Claude Code as an operational tool, not just a development tool. In a Githubified NanoClaw, when something goes wrong with a GitHub Actions run, the agent could be asked — through an issue comment — to diagnose the problem by reading the workflow logs, just as the local version is asked to read local logs.

### 5. Per-Group Memory Isolation

Each conversation group in NanoClaw gets its own isolated context:

```
groups/
├── main/
│   └── CLAUDE.md          # Admin group memory (read/write)
├── global/
│   └── CLAUDE.md          # Shared memory (read by all, written by main)
├── family-chat/
│   └── CLAUDE.md          # Isolated group memory
└── work-team/
    └── CLAUDE.md          # Isolated group memory
```

Groups cannot see each other's conversation history. The main group (self-chat) has admin privileges: it can write to global memory, schedule tasks for other groups, and manage all groups.

This pattern maps to GitHub Issues with labels or separate repositories: each "group" becomes an issue label or a dedicated repo, with memory stored in committed Markdown files. The permission model (main = admin, others = restricted) maps to GitHub's existing role system.

### 6. Single Pipeline Architecture

NanoClaw's message flow is a single pipeline:

```
Channels → SQLite → Polling loop → Container (Claude Agent SDK) → Response
```

There are no message queues, no event buses, no service meshes. The orchestrator (`src/index.ts`) polls for new messages, routes them through a per-group queue with a global concurrency limit, spawns a container for each invocation, and routes the response back.

For Githubification, this pipeline translates directly:

```
GitHub Issue comment → workflow trigger → Actions runner (Claude Agent SDK) → Issue reply + git commit
```

The polling loop becomes a webhook. The SQLite queue becomes the Actions job queue. The container becomes the runner. The channel-specific response routing becomes an issue comment.

### 7. The Skills Engine as Merge Infrastructure

The `skills-engine/` directory is a full programmatic skill management system with:

- **Three-way merge** — `git merge-file` against a shared base for code files
- **Structured operations** — deterministic handling of `package.json`, Docker Compose, `.env` files
- **Shared resolution cache** — `.nanoclaw/resolutions/` with hash-enforced pre-computed conflict resolutions
- **State tracking** — `.nanoclaw/state.yaml` records every applied skill, file hashes, and custom patches
- **Replay** — given `state.yaml`, reproduce the exact installation on a fresh machine with no AI
- **Uninstall as replay** — remove a skill by replaying all remaining skills from clean base
- **Rebase** — flatten accumulated layers into a clean starting point

This engine is more sophisticated than any Githubification installer seen so far. It solves the hard problem of composability: how do you combine multiple modifications to a shared codebase, keep them working across updates, and make them reversible? The answer — standard git merge mechanics with AI as a fallback — is a pattern the Githubification ecosystem should adopt for composing Githubification layers themselves.

### 8. Minimal Dependency Footprint

NanoClaw's `package.json` lists six runtime dependencies:

```json
{
  "dependencies": {
    "better-sqlite3": "^11.8.1",
    "cron-parser": "^5.5.0",
    "pino": "^9.6.0",
    "pino-pretty": "^13.0.0",
    "yaml": "^2.8.2",
    "zod": "^4.3.6"
  }
}
```

SQLite for state, cron parsing for scheduled tasks, pino for logging, YAML for skill manifests, Zod for validation. Everything else — the Claude Agent SDK, browser automation, channel libraries — is installed inside the container or added by skills.

This minimal footprint means the Githubification layer inherits almost no dependency management burden. Compare this to Githubifying a project with 70+ dependencies where version conflicts, native module compilation, and platform-specific binaries create installation complexity in the GitHub Actions environment.

### 9. Comprehensive Test Coverage

NanoClaw includes co-located tests for its core modules:

- `container-runner.test.ts` — container spawning logic
- `container-runtime.test.ts` — Docker / Apple Container abstraction
- `db.test.ts` — SQLite operations
- `formatting.test.ts` — message formatting
- `group-folder.test.ts` — group directory management
- `group-queue.test.ts` — per-group concurrency control
- `ipc-auth.test.ts` — IPC authorization
- `routing.test.ts` — message routing
- `task-scheduler.test.ts` — scheduled tasks

Tests run via `vitest`. The skills engine has its own test suite in `skills-engine/__tests__/`. This coverage means a Githubification layer can verify that its changes don't break the agent's core functionality — a critical property when wrapping an existing agent.

### 10. Explicit Security Documentation

`docs/SECURITY.md` provides a complete security model: trust levels (trusted main group vs. untrusted non-main groups vs. sandboxed container agents), security boundaries (container isolation, mount security, session isolation, IPC authorization, credential handling), privilege matrices, and an architecture diagram showing the flow from untrusted zone through host process to sandboxed container.

This level of security documentation is rare in AI agent projects and essential for Githubification. When the agent's execution environment changes from "local machine with containers" to "GitHub Actions runner," the security model must be re-evaluated. NanoClaw's explicit documentation makes that re-evaluation tractable — you can trace each security boundary and determine how it maps (or doesn't) to the GitHub Actions environment.

---

## What NanoClaw Teaches That Claw and OpenClaw Don't

The `lesson-from-claw.md` document in this directory shows how an agent born native to GitHub — `.github-claw/` — achieves the purest form of Githubification. The `lesson-from-openclaw.md` shows how a large, complex agent can be wrapped with a Githubification layer. NanoClaw occupies a third position: **a local agent deliberately designed for wrappability.**

1. **Complexity is the cost of Githubification.** OpenClaw's `.GITOPENCLAW/` layer is complex because OpenClaw is complex. GitClaw's `.github-claw/` layer is simple because it was born on GitHub. NanoClaw proves there's a middle path: a powerful agent that is simple enough that a Githubification layer would be nearly trivial. The wrapping cost is proportional to what you're wrapping.

2. **Skills are the contribution model Githubification needs.** Both GitClaw and OpenClaw accept features as code changes. NanoClaw rejects this entirely — contributions are skills that transform the codebase. Githubification itself could be a skill: `/githubify` transforms a local NanoClaw installation into one that runs on GitHub Actions. This is more composable than a monolithic Githubification folder.

3. **The skills engine solves the composition problem.** When multiple Githubification patterns need to coexist in a single repo (e.g., a GitClaw issue agent alongside a NanoClaw scheduled task agent), the conflict resolution and state tracking from NanoClaw's skills engine — three-way merge, shared resolution cache, deterministic replay — provides the infrastructure for composing them.

4. **AI-native agents need less scaffolding.** GitClaw's lifecycle scripts handle environment validation because the agent doesn't know it's on GitHub. OpenClaw's preflight checks validate configuration because the agent has complex requirements. NanoClaw's `/debug` skill means the agent can diagnose its own problems. In a Githubified NanoClaw, the preflight step could be "ask the agent if it's healthy" rather than a hand-written validation script.

5. **Container isolation translates directly.** NanoClaw already runs agents in containers. GitHub Actions already provides isolated runners. The security model — mount restrictions, credential filtering, non-root execution — maps almost 1:1 to what GitHub Actions enforces naturally. There is no security boundary to reinvent.

6. **Per-group memory becomes per-issue memory.** NanoClaw's `groups/*/CLAUDE.md` pattern — isolated memory files committed to the filesystem — is exactly the pattern Githubified repos use for per-issue session state. The mapping is natural: `groups/main/CLAUDE.md` → `.githubification/state/issues/main.md`.

---

## Lessons for Any Type 1 Githubification

1. **Design for AI legibility.** If the agent is small enough for an AI to understand in a single context window, the Githubification layer will be small too. Complexity in the agent creates complexity in the wrapper. NanoClaw's 35,000-token constraint is aspirational for any agent that intends to be Githubified.

2. **Separate the channel from the logic.** NanoClaw's channel registry pattern — channels self-register at startup, the orchestrator is channel-agnostic — means replacing WhatsApp with GitHub Issues is a channel swap, not an architectural change. Design agents so the messaging layer is pluggable.

3. **Use container isolation that maps to the target.** If you know the agent will eventually run on GitHub Actions, design its local isolation model to match. NanoClaw's ephemeral containers with mount restrictions and credential filtering are structurally identical to GitHub Actions runners with secret injection.

4. **Make Githubification a skill, not a layer.** Instead of a `.GITAGENT/` folder with lifecycle scripts, consider a Claude Code skill that teaches an AI how to add GitHub Actions support. The result is cleaner code, specific to the user's needs, and composable with other skills.

5. **Build the merge infrastructure.** Multiple skills, multiple Githubification patterns, and multiple agents will need to coexist in the same repo. NanoClaw's skills engine — three-way merge, resolution caching, state tracking, deterministic replay — is the infrastructure for composing these changes without conflicts.

6. **Keep dependencies minimal.** Every runtime dependency the agent brings is a dependency the GitHub Actions workflow must install. Six dependencies install in seconds. Seventy dependencies install in minutes and may fail on platform mismatches. Minimize the footprint, and the Githubification layer minimizes itself.

7. **Document the security model explicitly.** When the execution environment changes (local → GitHub Actions), security boundaries must be re-evaluated. An explicit security document with trust levels, boundary definitions, privilege matrices, and architecture diagrams makes this re-evaluation possible. Without it, the Githubification layer is guessing.

8. **Test the agent, not just the wrapper.** NanoClaw's co-located test suite (`vitest`) means the Githubification layer can run the agent's own tests in the GitHub Actions environment to verify nothing broke. Structure tests so they're environment-agnostic — they should pass locally and on a runner.

9. **AI-native means less lifecycle.** An agent that can diagnose its own problems (via `/debug`), configure itself (via `/setup`), and customize itself (via `/customize`) needs less hand-written lifecycle management in the Githubification layer. The guard → validate → indicate → execute pipeline can be shortened to guard → execute if the agent handles its own validation.

10. **The contribution model is the scaling model.** If contributors add features, the codebase grows and the Githubification layer must grow with it. If contributors add skills, the base codebase stays fixed and the Githubification layer stays fixed. NanoClaw's "skills over features" philosophy is the scaling model for maintainable Githubification.

---

## Summary

`japer-technology/github-nanoclaw` demonstrates that the easiest agent to Githubify is one that was designed to be simple. A single Node.js process, six runtime dependencies, container-based isolation, per-group memory in committed files, and a channel-agnostic architecture — every design decision makes the mapping from local execution to GitHub-as-infrastructure nearly mechanical. The channel registry becomes GitHub Issues. The SQLite state becomes git commits. The container runner becomes a GitHub Actions step. The `.env` credentials become repository secrets.

What makes NanoClaw uniquely instructive is not just its simplicity but its extensibility model. The skills architecture — Claude Code skills that programmatically transform the codebase, backed by a three-way merge engine with resolution caching and deterministic replay — solves the composition problem that Githubification will inevitably face. When Githubification itself is a skill rather than a monolithic wrapper, it becomes composable, reversible, and maintainable.

Where GitClaw proves that agents born on GitHub need no wrapping, and OpenClaw proves that complex agents can be wrapped, NanoClaw proves that **agents designed for simplicity make Githubification nearly free**. The cost of wrapping is proportional to the complexity of what you wrap. Design the agent for legibility, and the Githubification layer writes itself.
