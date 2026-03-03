# Lesson Consolidation

### What six case studies teach us about Githubification — the unified playbook

---

## The Six Subjects

This document consolidates the lessons from six Githubified repositories, each representing a different point on the spectrum of agent complexity, origin, and Githubification strategy:

| # | Repository | Agent | Strategy | Key Lesson |
|---|-----------|-------|----------|------------|
| 1 | `japer-technology/github-openclaw` | OpenClaw — 30+ tools, semantic memory, browser automation | **Wrapping** | A complex agent can be Githubified without modifying a single line of its source |
| 2 | `japer-technology/github-claw` | GitClaw — pi-based, born for GitHub | **Native** | The purest Githubification is an agent that was never meant to run anywhere else |
| 3 | `japer-technology/github-minimum-intelligence` | GMI — pi-based, born for GitHub | **Native** | When the agent IS the Githubification layer, the architecture collapses to its simplest form |
| 4 | `japer-technology/github-agent0` | Agent Zero — 19+ tools, Flask server, FAISS, Docker | **Substitution** | When wrapping is infeasible, substitute a GitHub-native agent alongside the codebase |
| 5 | `japer-technology/github-agenticana` | Agenticana — 20 specialist agents, swarm orchestration | **Transformation** | Multi-agent systems need a routing layer that doesn't exist in single-agent Githubification |
| 6 | `japer-technology/github-microclaw` | MicroClaw — Rust binary, 14 platforms, 20+ tools | **Channel addition** | When the agent is channel-agnostic, Githubification is just adding another adapter |

Together, these six cases span the full range: from a single-dependency agent born on GitHub to a 20-agent orchestration framework designed for VS Code, from a simple TypeScript wrapper to a compiled Rust binary with zero runtime dependencies.

---

## The One Invariant

Across all six repositories — regardless of language, complexity, origin, or strategy — the same four GitHub primitives serve the same four roles:

| GitHub Primitive | Role | Universal Across All Six |
|---|---|---|
| **GitHub Actions** | Compute | The runner that executes the agent — whether it's a TypeScript lifecycle script, a Python CLI, or a compiled Rust binary |
| **Git** | Storage and memory | Sessions, conversations, decisions, and state are committed — JSONL, JSON, Markdown, or SQLite |
| **GitHub Issues** | User interface | Each issue is a conversation thread — the agent reads comments, posts replies, and remembers across sessions |
| **GitHub Secrets** | Credential store | LLM API keys, GitHub App credentials, and tokens — no hardcoded secrets |

This mapping is the invariant of Githubification. It holds for a 1-dependency native agent (GMI) and for a 54-dependency Python framework that can't even run on Actions (Agent Zero). It holds for a single agent and for twenty. The primitives don't change. What changes is the strategy for connecting the agent to them.

---

## The Five Strategies

The six case studies reveal five distinct strategies for Githubification, ordered by the relationship between the agent and the GitHub runtime:

### Strategy 1 — Native (GitClaw, GMI)

**The agent was designed for GitHub from the start.**

There is no separate agent to wrap. The `.github-claw/` or `.github-minimum-intelligence/` folder IS the agent. The lifecycle scripts don't translate between GitHub and some external runtime — they speak GitHub natively. The result is the simplest architecture: a two-or-three-step lifecycle pipeline, a single runtime dependency (`pi-coding-agent`), and zero impedance mismatch.

| Characteristic | Detail |
|---|---|
| **Agent origin** | Built for GitHub from scratch |
| **Githubification folder** | IS the agent |
| **Lifecycle complexity** | 2–3 steps |
| **Runtime dependencies** | 1 (`pi-coding-agent`) |
| **New code required** | None — the agent IS the Githubification |

**When to use:** You're building a new agent and can choose its runtime from the start. Design it for GitHub. The lifecycle is simpler, the configuration is cleaner, and there are fewer failure modes.

### Strategy 2 — Wrapping (OpenClaw)

**The agent exists; Githubification wraps it without modification.**

The `.GITOPENCLAW/` folder sits alongside the agent's source code and orchestrates when and how the agent runs on GitHub Actions. The agent doesn't know it's running on GitHub — the lifecycle scripts translate between GitHub primitives and the agent's native interfaces.

| Characteristic | Detail |
|---|---|
| **Agent origin** | External project, ported to GitHub |
| **Githubification folder** | Alongside the agent source |
| **Lifecycle complexity** | 5-step pipeline |
| **Runtime dependencies** | The agent's full dependency tree (30+ tools, vector DB, etc.) |
| **New code required** | Orchestration layer (TypeScript lifecycle scripts) |

**When to use:** The agent already exists and has a user base. Wrapping preserves the upstream project and avoids forking. Upstream updates can be pulled without conflicts because the agent source is untouched.

### Strategy 3 — Substitution (Agent Zero)

**The agent's runtime conflicts with GitHub Actions; substitute a different, GitHub-native agent.**

Agent Zero requires a persistent HTTP server, WebSocket streaming, FAISS databases, and Docker sandboxing — all incompatible with GitHub Actions' ephemeral execution model. Instead of forcing a brittle partial wrap, the `.issue-intelligence/` folder deploys a different agent (the pi coding agent) that is GitHub-native, with Agent Zero's entire codebase available as read context.

| Characteristic | Detail |
|---|---|
| **Agent origin** | External project, incompatible runtime |
| **Githubification folder** | Alongside the agent source |
| **Lifecycle complexity** | 3-step pipeline (same as native) |
| **Runtime dependencies** | 1 (`pi-coding-agent`) |
| **New code required** | The substituted agent's configuration and context |

**When to use:** The agent requires persistent processes, inbound networking, GPU, or other capabilities that GitHub Actions cannot provide. Don't force-fit. Deploy a capable GitHub-native agent that can read, explain, analyze, and modify the original agent's codebase.

### Strategy 4 — Transformation (Agenticana)

**The agent was designed for a different runtime; Githubification transforms its interaction model.**

Agenticana's 20 specialist agents were designed for VS Code and CLI. Githubification must create a routing layer that maps GitHub events (issue opened, label applied, comment created) to the correct specialist agent. This dispatch layer is new infrastructure — not just a wrapper.

| Characteristic | Detail |
|---|---|
| **Agent origin** | Built for VS Code/CLI, ported to GitHub |
| **Githubification folder** | Primarily analysis, not yet implementation |
| **Lifecycle complexity** | Requires multi-agent dispatch |
| **Runtime dependencies** | Python scripts + Node.js MCP |
| **New code required** | Agent router, swarm orchestration, state sync |

**When to use:** The agent's interaction model (synchronous, interactive, local) is fundamentally different from GitHub's (asynchronous, event-driven, remote). The transformation must bridge this gap while preserving the specialization that makes the agents valuable.

### Strategy 5 — Channel Addition (MicroClaw)

**The agent already supports multiple platforms; GitHub becomes just another channel.**

MicroClaw's channel-agnostic architecture (Telegram, Discord, Slack, Feishu, Web, etc.) means adding GitHub Issues is adding a new adapter to an existing multi-channel system. The agent orchestrates itself — no external lifecycle scripts needed.

| Characteristic | Detail |
|---|---|
| **Agent origin** | External project with channel-agnostic design |
| **Githubification folder** | Design documents + minimal adapter code |
| **Lifecycle complexity** | Workflow → binary → response |
| **Runtime dependencies** | 0 (compiled Rust binary) |
| **New code required** | ~500 lines (channel adapter + CLI subcommand) |

**When to use:** The agent already has a platform adapter architecture. If there's a clean separation between "how it receives input" and "how it processes it," Githubification becomes an adapter problem, not a rewrite.

---

## Strategy Selection Guide

```
Does the agent exist yet?
├── No → Build it native (Strategy 1: Native)
└── Yes
    ├── Can it run on GitHub Actions?
    │   ├── No (requires persistent server, GPU, etc.)
    │   │   └── Deploy a GitHub-native agent alongside it (Strategy 3: Substitution)
    │   └── Yes
    │       ├── Does it have a multi-channel/adapter architecture?
    │       │   ├── Yes → Add GitHub as a channel (Strategy 5: Channel Addition)
    │       │   └── No
    │       │       ├── Is it a single agent or multiple?
    │       │       │   ├── Single → Wrap it (Strategy 2: Wrapping)
    │       │       │   └── Multiple → Transform with routing layer (Strategy 4: Transformation)
    │       │       └── Is the interaction model compatible?
    │       │           ├── Yes → Wrap it (Strategy 2: Wrapping)
    │       │           └── No → Transform it (Strategy 4: Transformation)
    │       └──
    └──
```

---

## The Universal Patterns

Across all six case studies, regardless of strategy, the same structural patterns recur. These are the building blocks of any Githubification.

### 1. The Lifecycle Pipeline

Every Githubified repo follows an ordered pipeline from event to response. The number of steps varies, but the structure is constant:

| Step | OpenClaw | GitClaw | GMI | Agent0 | Agenticana | MicroClaw |
|------|----------|---------|-----|--------|------------|-----------|
| **Guard** | Sentinel file (`ENABLED.md`) | Sentinel file | Workflow auth step | Sentinel file | — (analysis phase) | Workflow auth step |
| **Validate** | Preflight script | — | — | — | — | — |
| **Indicate** | 👀 reaction | 👀 reaction | 🚀 reaction | 👀 reaction | — | 🚀 reaction |
| **Execute** | Agent orchestrator | Agent orchestrator | Agent orchestrator | Agent orchestrator | — | Binary + CLI subcommand |
| **Commit** | Git push with retry | Git push with retry | Git push with retry | Git push with retry | — | Git push with retry |

The pipeline can be as lean as two files (GMI) or as structured as five steps (OpenClaw). The invariant is the pattern itself: guard → indicate → execute → commit. Each step is discrete, independently testable, and independently failable.

**Consolidation:** Start with the minimal pipeline (guard → indicate → execute → commit). Add validation steps only when the environment is unpredictable — native agents need less validation because they were designed for the environment they run in.

### 2. Fail-Closed Security

Two approaches, both valid:

| Approach | Used By | Mechanism | Kill Switch |
|----------|---------|-----------|-------------|
| **Sentinel file** | OpenClaw, GitClaw, Agent0 | A Markdown file (`*-ENABLED.md`) that must exist; lifecycle script checks and exits if missing | `git rm` the file |
| **Workflow authorization** | GMI, MicroClaw | Workflow step queries GitHub API for actor's permission level; rejects if not admin/maintain/write | GitHub's permission model |

Sentinel files give a repo-level kill switch that any collaborator can use. Workflow authorization leverages GitHub's existing access control without requiring file management. Choose based on your threat model: sentinel files for explicit opt-in/opt-out, workflow auth for permission-based control.

Agent Zero adds a third pattern: **heart-gating** — requiring a ❤️ reaction on issues before the agent processes them. This is a lightweight moderation mechanism for public repos with high traffic.

**Consolidation:** Every Githubified repo must be fail-closed. Pick one of the three mechanisms (sentinel file, workflow auth, or heart-gating) based on who should control agent activation and how granular that control needs to be.

### 3. Issue-Driven Conversation

The mapping is identical across all six repos:

```
Issue #N → state/issues/N.json → state/sessions/<timestamp>.jsonl
```

When a user comments on issue #N weeks later, the agent loads the linked session file and resumes with full context. No database, no session cookies — just git. This pattern is so fundamental that it appears unchanged in every case study.

**Consolidation:** Issue-driven conversation is not a pattern to choose — it's a pattern to implement. The mapping from issue number to session file to conversation transcript is the canonical state management model for any Githubified repo.

### 4. Git as Memory

Every case study commits agent state to git. The formats vary:

| Repo | Session Format | Memory Format | State Format |
|------|---------------|---------------|-------------|
| OpenClaw | JSONL | Append-only `memory.log` with `merge=union` | JSON issue mappings |
| GitClaw | JSONL | JSONL sessions | JSON issue mappings |
| GMI | JSONL | Append-only log + skills | JSON issue mappings |
| Agent0 | JSONL | JSONL sessions | JSON issue mappings |
| Agenticana | — (design phase) | ReasoningBank (JSON with embeddings) | — |
| MicroClaw | SQLite | Structured + file memory | SQLite + readable exports |

MicroClaw's SQLite approach introduces a pragmatic variation: commit the binary database alongside human-readable companion exports. The tradeoff is opaque binary diffs in exchange for zero changes to the agent's persistence layer.

**Consolidation:** Commit everything a human would want to review. Sessions, responses, decisions, and memory go in git. Caches, databases (unless the agent requires them), and dependencies go in `.gitignore`. If the agent uses a binary format like SQLite, pair it with human-readable exports for auditability.

### 5. Concurrency Resilience

Every repo that runs on GitHub Actions faces the same problem: multiple issues can trigger the agent simultaneously, and multiple git pushes will conflict. The solution is consistent:

1. **Per-issue concurrency groups** — `agent-${{ github.repository }}-issue-${{ github.event.issue.number }}` with `cancel-in-progress: false`
2. **Git push retry loop** — 10 attempts with escalating backoff, using `git pull --rebase -X theirs` on conflicts

This is a solved problem. The implementation is nearly identical across OpenClaw, GitClaw, GMI, Agent0, and MicroClaw.

**Consolidation:** Copy the concurrency group and retry loop from any existing implementation. This is infrastructure, not design — it should be identical in every Githubified repo.

### 6. Self-Contained Folder

Every Githubified repo follows the same containment principle: everything the agent needs lives in one folder.

| Repo | Folder | Contains |
|------|--------|----------|
| OpenClaw | `.GITOPENCLAW/` | Lifecycle, config, state, installer, tests, docs |
| GitClaw | `.github-claw/` | Lifecycle, config, state, installer, tests, help, docs |
| GMI | `.github-minimum-intelligence/` | Lifecycle, config, state, installer, skills, docs |
| Agent0 | `.issue-intelligence/` | Lifecycle, config, state, installer, help, docs |
| Agenticana | `.github-agenticana/` | Analysis, specifications, design documents |
| MicroClaw | `.github-microclaw/` | Design documents, architecture mapping |

The folder is the product. It can be copied, version-controlled, backed up, and deleted as a single unit. The boundary between "agent" and "not agent" is a directory listing. Zero dependencies on files outside the folder (except `.github/workflows/` and `.github/ISSUE_TEMPLATE/`, which are created by the installer).

**Consolidation:** The agent folder must be self-contained and droppable into any repo without modification. If it requires changes to the host's existing files, the installation boundary is leaking. The installer creates the workflow and issue templates in `.github/` — that's the only sanctioned external touch point.

### 7. Personality and Identity

Every case study provides a mechanism for agent identity, with increasing sophistication:

| Repo | Identity Mechanism | Identity File |
|------|-------------------|---------------|
| OpenClaw | Configured in `AGENTS.md` | `AGENTS.md` |
| GitClaw | Hatching conversation → `AGENTS.md` | `AGENTS.md` |
| GMI | Hatching conversation → `AGENTS.md` + `user.md` | `AGENTS.md` |
| Agent0 | Hatching conversation → `AGENTS.md` | `AGENTS.md` |
| Agenticana | 20 YAML specs + MD instructions per agent | Per-agent YAML + MD |
| MicroClaw | `SOUL.md` + `AGENTS.md` + per-chat overrides | Three-tier: global → channel → conversation |

GitClaw introduced **personality hatching** — a guided conversation where the user and agent collaboratively define the agent's name, nature, and vibe. This pattern was adopted by GMI and Agent0. MicroClaw extends it with a three-tier identity system (global → channel → conversation).

**Consolidation:** Agent identity is not optional — it's what makes users adopt and trust the agent. At minimum, provide an `AGENTS.md` file. For better user experience, provide a hatching conversation. For multi-context agents, consider per-conversation identity overrides.

### 8. Installer as Infrastructure

Every case study that has reached implementation includes an installer:

| Repo | Installer | Mechanism |
|------|-----------|-----------|
| OpenClaw | `GITOPENCLAW-INSTALLER.yml` | GitHub Actions workflow that creates a PR |
| GitClaw | `github-claw-INSTALLER.ts` | TypeScript script run with Bun |
| GMI | `MINIMUM-INTELLIGENCE-INSTALLER.ts` + `setup.sh` | Script + curl-pipe-bash + GitHub App |
| Agent0 | `ISSUE-INTELLIGENCE-INSTALLER.ts` | TypeScript script run with Bun |
| MicroClaw | — (design phase) | Planned: binary download from releases |

The installer creates `.github/workflows/` and `.github/ISSUE_TEMPLATE/` entries, copies templates, and initializes configuration. GMI's `setup.sh` adds an important detail: it **removes the `state/` directory and resets identity files** after downloading, ensuring new installations don't inherit the source repo's agent personality or conversation history.

**Consolidation:** The installer is part of the product, not an afterthought. It must be idempotent (skip files that already exist), self-contained (no external dependencies beyond the agent's runtime), and state-resetting (new installations start clean).

### 9. Documentation as Architecture

Every case study includes documentation beyond a README, with increasing depth:

| Repo | Documentation | Purpose |
|------|--------------|---------|
| OpenClaw | `analysis/`, capability comparisons | Design reasoning |
| GitClaw | `help/`, `THE-REPO-IS-THE-MIND.md` | Operational docs + design theory |
| GMI | `docs/` (DEFCON levels, Four Laws, foundational questions, security assessment) | Governance framework |
| Agent0 | Feasibility assessment, runtime catalog, design theory extraction | Honest constraint analysis |
| Agenticana | Eight analysis documents studying reference architecture | Research-first approach |
| MicroClaw | Five design documents mapping GMI to MicroClaw | Architecture mapping |

GMI's documentation corpus is the most architecturally significant: DEFCON readiness levels, the Four Laws of AI, foundational questions (What? Who? When? Where? How? How Much?), security assessment, and incident response procedures. These provide a governance framework adoptable by any Githubified repo.

Agent Zero's documentation is the most honest: a component-by-component feasibility assessment that concludes "we can't fully Githubify this, and here's exactly why."

Agenticana's documentation is the most analytical: eight documents studying the reference architecture before writing any code.

**Consolidation:** Documentation is infrastructure. At minimum: a README, a quickstart, and operational help (install, configure, enable, disable, uninstall). For complex agents: add feasibility assessment, architecture mapping, and design theory. For governance: adopt GMI's DEFCON levels and foundational questions as templates.

### 10. Test the Structure, Not the AI

The repos that include tests (OpenClaw, GitClaw, Agent0) all follow the same principle: test that the Githubification layer is correctly structured — files exist, config is valid, workflows are in place, lifecycle scripts parse correctly. They do not test LLM responses.

**Consolidation:** Structure is deterministic and testable; AI output is not. Write tests that validate: required files exist, configuration files parse correctly, workflow YAML is valid, lifecycle scripts import without errors, and state directories are initialized. Do not test LLM responses.

---

## The Spectrum of Complexity

The six case studies reveal a spectrum from minimal to maximal Githubification complexity:

```
Simplest                                                               Most Complex
   │                                                                        │
   ▼                                                                        ▼
  GMI          GitClaw        Agent0        OpenClaw      MicroClaw     Agenticana
   │              │              │              │              │              │
 2 files       3 steps      Substitution   5-step wrap   Channel add   20-agent route
 1 dep         1 dep         1 dep         30+ deps       0 deps        Multi-runtime
 Native        Native        Substitute     Wrapped       Adapter       Transform
```

Yet the four primitives (Actions, Git, Issues, Secrets) remain constant across the entire spectrum. The complexity gradient is not in what GitHub provides — it's in the mapping between the agent and those primitives.

### What Scales

- The **four primitives** — universal at every complexity level
- The **lifecycle pipeline** — guard → indicate → execute → commit, at every scale
- **Issue-driven conversation** — one issue = one thread, from 1 agent to 20
- **Git as memory** — sessions committed, from JSONL to SQLite
- **Concurrency resilience** — per-issue groups + retry loop, always

### What Changes

- **Routing** — not needed for single agents, critical for multi-agent systems (Agenticana)
- **Lifecycle depth** — 2 files for native, 5 steps for wrapped, dispatch layer for transformation
- **Dependency surface** — 0 (compiled binary) to 54 (Python framework)
- **State format** — human-readable files to binary databases
- **Identity complexity** — single `AGENTS.md` to three-tier personality system
- **New code required** — 0 lines (native) to thousands (transformation)

---

## Cross-Cutting Lessons

### From OpenClaw: The Discipline of Non-Modification

> **Don't fork, don't patch — wrap.**

The `.GITOPENCLAW/` folder doesn't change a single line of OpenClaw code. Every file outside the Githubification folder remains exactly as the upstream project ships it. This means upstream updates can be pulled without conflicts. This discipline — Githubification as addition, never modification — is the foundation of the wrapping strategy and a constraint worth preserving in every approach.

### From GitClaw: The Elegance of Native Design

> **The folder is the product.**

When the agent is designed for GitHub from the start, there is no impedance mismatch. The agent's input is an issue comment. Its output is an issue reply and a git commit. Its memory is the repository. Its compute is a runner. Every design decision flows from that premise. The result is radical simplicity: one dependency, three lifecycle steps, zero translation.

GitClaw also introduces composition over inheritance: the agent folder should have zero dependencies on files outside itself. It should be droppable into any repo without modification. Installation is composition — the repository HAS an agent. Forking is inheritance — the repository IS the agent.

### From GMI: The Governance Framework

> **Document the philosophy, not just the code.**

GMI's DEFCON readiness levels, Four Laws of AI, foundational questions, and security assessment provide a governance framework that transcends any single agent. These documents answer questions that every Githubified repo should answer: What is this? Who controls it? When does it act? Where does it operate? How does it work? How much does it cost?

GMI also proves that native agents can be distributed: the `setup.sh` + curl-pipe-bash installation path makes any repository a one-command install away from having an AI agent. The critical detail: reset state and identity on distribution.

### From Agent Zero: The Honesty of Limits

> **If full Githubification isn't possible, say so clearly and explain why.**

Agent Zero is the first case where the honest answer is: the agent's core experience cannot be replicated on GitHub Actions. Rather than attempting a brittle partial wrapping, the project substitutes a different agent and documents exactly why. The feasibility assessment — evaluating each component against GitHub Actions' constraints — is a template for any non-trivial Githubification.

Agent Zero also demonstrates that the Githubification process itself produces valuable analysis. Even when wrapping fails, the deep reading of the agent's architecture produces design theories applicable far beyond GitHub Actions.

### From Agenticana: The Research Phase

> **Complex Githubification benefits from analysis before implementation.**

Agenticana's eight-document analysis corpus studies the reference architecture, maps each pattern to the target system, and documents the design before writing any lifecycle scripts. This analysis-first approach is especially valuable when the source system's architecture diverges significantly from the GitHub-native pattern.

Agenticana introduces multi-agent routing as a first-class concern: when the repo contains more than one agent, Githubification must introduce dispatch logic that maps GitHub events to the correct agent. Issue labels become routing keys. The workflow reads the label and invokes the corresponding agent script. This is new infrastructure that doesn't exist in single-agent systems.

### From MicroClaw: The Channel-Agnostic Ideal

> **If your agent supports multiple platforms through adapters, adding GitHub is just adding another adapter.**

MicroClaw's Githubification is ~500 lines of new code — a channel adapter, a CLI subcommand, a workflow, and a config file. Everything else — the agent loop, the LLM layer, the tool registry, the memory system — remains unchanged. This is possible because MicroClaw was designed for platform independence.

MicroClaw also introduces new patterns: compiled binary distribution (download from releases, zero dependency installation), SQLite-in-git persistence (binary database + readable companion exports), and Docker-optional sandboxing for defense-in-depth.

---

## The Consolidated Playbook

### For Any Githubification

1. **Start with feasibility.** Evaluate the agent's runtime requirements against GitHub Actions' constraints: no inbound networking, ephemeral runners, 6-hour maximum job duration, no GPU on standard runners, pay-per-minute compute. If any core component is infeasible, consider substitution over wrapping.

2. **Choose the right strategy.** Native if building new. Wrapping if the agent exists and fits Actions. Substitution if it doesn't fit. Transformation if multi-agent. Channel addition if the agent is already platform-agnostic.

3. **Map to the four primitives.** Actions = compute, Git = memory, Issues = UI, Secrets = credentials. This mapping is the invariant. Design everything else around it.

4. **Implement the lifecycle pipeline.** Guard → indicate → execute → commit. Start minimal (2 files for native) and add steps only when needed (validation for wrapped, routing for multi-agent).

5. **Make it fail-closed.** Nothing runs unless explicitly enabled. Sentinel file, workflow authorization, or heart-gating — pick one, implement it, and make it the first thing that runs.

6. **Commit auditable state.** Sessions, responses, decisions, and memory go in git. Caches, databases, and dependencies go in `.gitignore`. If git is the memory, treat it with the same care as a production database.

7. **Handle concurrency.** Per-issue concurrency groups with `cancel-in-progress: false`. Git push retry loop with 10 attempts and escalating backoff. This is infrastructure — copy it, don't redesign it.

8. **Contain everything in one folder.** The agent folder is the product. Zero dependencies on files outside it (except `.github/workflows/` and `.github/ISSUE_TEMPLATE/`, created by the installer). Copy the folder in, delete the folder out.

9. **Build the installer.** Idempotent, self-contained, state-resetting. It creates workflows, issue templates, and initializes configuration. New installations must not inherit the source repo's conversation history or agent identity.

10. **Give the agent an identity.** `AGENTS.md` at minimum. Personality hatching for better user experience. Per-conversation overrides for multi-context agents. Identity is what makes users adopt and trust the agent.

11. **Test the structure.** Files exist, config parses, workflows are valid, scripts import cleanly. Don't test LLM output. Structure is deterministic; AI is not.

12. **Document the reasoning.** README, quickstart, operational help at minimum. Feasibility assessment, architecture mapping, and design theory for complex agents. Governance framework (DEFCON levels, security assessment) for production use.

### For Specific Strategies

**If native:**
- Design for GitHub from day one — the lifecycle will be simpler and the configuration cleaner
- Lean on an existing agent runtime (like `pi-coding-agent`) — don't reimplement LLM interaction, tool execution, or session management
- Skills as Markdown — make capabilities readable, composable, and user-editable without touching code

**If wrapping:**
- Don't modify the agent source — Githubification is an addition, not a modification
- Separate agent state from agent source — all runtime data in `state/`, agent code is read-only context
- Include a preflight validation step — the runtime environment is less predictable than for native agents

**If substituting:**
- Be honest about what can't run on Actions and why
- Use the original agent's source code as domain context — the codebase is a knowledge base
- The substituted agent should still provide genuine value: code explanation, architecture analysis, modification assistance

**If transforming:**
- Analyze before you build — study the reference architecture, map each pattern, document the design
- Create the routing layer explicitly — labels as dispatch keys, event types as triggers
- Adapt the interaction model — synchronous/interactive → asynchronous/event-driven

**If adding a channel:**
- The agent should orchestrate itself — the workflow's job is authorization, setup, and state commit
- CLI subcommands enable one-shot mode — process one event and exit
- If the agent is compiled, distribute as a pre-built binary from releases

---

## Summary

Six repositories. Five strategies. Four primitives. One pattern.

Githubification turns GitHub from a place where code is stored into a place where code is **run**. The method varies — native agents collapse the Githubification layer into the agent itself, wrapping preserves existing agents without modification, substitution provides value when wrapping fails, transformation bridges the gap between local-first and GitHub-native interaction models, and channel addition leverages existing platform-agnostic architectures.

What doesn't vary is the destination: an AI agent that runs on GitHub Actions, converses through Issues, persists through Git, and authenticates through Secrets. The four primitives are universal. The lifecycle pipeline is universal. Issue-driven conversation is universal. Git as memory is universal. Concurrency resilience is universal. The self-contained folder is universal.

The complexity gradient — from GMI's two files to Agenticana's twenty-agent routing layer — is not in what GitHub provides. It's in the distance between the agent's native architecture and GitHub's execution model. The closer the agent is to GitHub's primitives, the thinner the Githubification layer. The farther away, the more infrastructure the layer must provide.

The consolidated lesson is this: **any AI agent can be Githubified, but how it's Githubified depends on what it is, where it came from, and how far it needs to travel to reach GitHub's four primitives.** The playbook above maps that journey for every starting point the case studies have revealed — and for any new starting point that follows the same patterns.
