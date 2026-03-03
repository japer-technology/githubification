# Lesson from GMI

### What `japer-technology/github-minimum-intelligence` teaches us about Type 1 — AI Agent Repo Githubification

---

## The Subject

[GitHub Minimum Intelligence](https://github.com/japer-technology/github-minimum-intelligence) is a repository-local AI framework that turns any GitHub repo into an interactive AI workspace. Powered by the [pi coding agent](https://github.com/badlogic/pi-mono), it uses GitHub Issues as a conversational interface, Git as persistent memory, GitHub Actions as compute, and GitHub Secrets as a credential store. A single folder — `.github-minimum-intelligence/` — contains the entire system: orchestrator, runtime configuration, session state, installer, documentation, and a modular skill system.

Unlike the OpenClaw case study, where Githubification was applied _to_ an existing AI agent, GitHub Minimum Intelligence was **designed from the ground up to live inside a repository**. The agent was never meant to run locally on a server — GitHub _is_ its native environment. This makes it a uniquely instructive example of Type 1 Githubification: the Githubification layer and the agent are the same thing.

This is a textbook example of **Type 1 — AI Agent Repo** Githubification, but with a twist: the agent was born Githubified.

---

## The Core Lesson

> **When the agent is designed for GitHub from the start, the Githubification layer collapses into the agent itself.**

In the OpenClaw case, Githubification wraps an external agent — the `.GITOPENCLAW/` folder sits alongside OpenClaw's source code and maps GitHub primitives to the agent's native interfaces. The agent doesn't know it's running on GitHub.

GitHub Minimum Intelligence takes a different path. There is no separate agent to wrap. The `.github-minimum-intelligence/` folder _is_ the product. The lifecycle scripts don't translate between GitHub and some external runtime — they speak GitHub natively. The result is a dramatically simpler architecture that still maps to the same four GitHub primitives:

| GitHub Primitive | Maps To |
|---|---|
| **GitHub Actions** | Compute — the runner that executes the agent workflow |
| **Git** | Storage and memory — sessions, conversations, and agent edits are committed |
| **GitHub Issues** | User interface — each issue is a conversation thread |
| **GitHub Secrets** | Credential store — LLM API keys, GitHub App credentials |

The agent knows it's running on GitHub. It was born there.

---

## Anatomy of the `.github-minimum-intelligence/` Folder

The entire system is self-contained:

```
.github-minimum-intelligence/
├── .pi/                                # Agent personality & skills config
│   ├── settings.json                   # LLM provider, model, thinking level
│   ├── APPEND_SYSTEM.md                # System prompt loaded every session
│   ├── BOOTSTRAP.md                    # First-run identity prompt (hatching)
│   └── skills/                         # Modular skill packages
│       ├── memory/                     # Long-term memory skill
│       └── skill-creator/              # Meta-skill for creating new skills
├── AGENTS.md                           # Agent identity — name, personality, vibe
├── PACKAGES.md                         # Dependency documentation
├── docs/                               # Comprehensive documentation
│   ├── index.md                        # Documentation index
│   ├── final-warning.md                # Usage precautions and governance
│   ├── incident-response.md            # Security incident procedures
│   ├── security-assessment.md          # Threat model and vulnerability review
│   ├── warning-blast-radius.md         # Capabilities audit
│   ├── the-four-laws-of-ai.md          # Governing laws
│   ├── the-repo-is-the-mind.md         # Architectural thesis
│   ├── question-{what,who,when,where,how,how-much}.md  # Foundational questions
│   ├── transition-to-defcon-{1..5}.md  # DEFCON readiness levels
│   └── analysis/                       # Design documents
├── install/                            # Installer and templates
│   ├── MINIMUM-INTELLIGENCE-INSTALLER.ts  # Setup script
│   ├── MINIMUM-INTELLIGENCE-AGENTS.md     # Default agent identity template
│   ├── github-minimum-intelligence-agent.yml   # Workflow template
│   ├── github-minimum-intelligence-chat.md     # Issue template (chat)
│   ├── github-minimum-intelligence-hatch.md    # Issue template (hatching)
│   ├── settings.json                   # Default LLM config
│   └── package.json                    # Installer dependencies
├── lifecycle/                          # Runtime scripts
│   ├── indicator.ts                    # Adds 🚀 reaction to show agent is working
│   └── agent.ts                        # Core orchestrator — runs the AI, posts replies
├── state/                              # Session history and issue mappings (git-tracked)
│   ├── issues/                         # Issue number → session file mappings
│   ├── sessions/                       # Conversation transcripts (JSONL)
│   └── user.md                         # User identity and preferences
├── logo.png                            # Project branding
├── bun.lock                            # Dependency lockfile
└── package.json                        # Runtime dependencies (one: pi-coding-agent)
```

Everything the agent needs to exist, run, and remember lives in this folder. There is no separate software to wrap — the folder _is_ the software.

---

## Key Patterns

### 1. Two-File Lifecycle

Where OpenClaw has a multi-step pipeline (enabled guard → preflight → indicator → agent), GMI compresses the lifecycle into just two TypeScript files:

| Step | File | Purpose |
|------|------|---------|
| 1 | `indicator.ts` | Add 🚀 reaction to show the agent is working |
| 2 | `agent.ts` | Run the AI agent, post the reply, commit state |

Authorization is handled by the workflow itself (a shell step that checks collaborator permissions via `gh api`), not by a separate lifecycle script. This keeps the TypeScript layer focused on agent logic.

The workflow step order is:

| # | Step | What Happens |
|---|------|------|
| 1 | Authorize | Shell — check collaborator permission via `gh api` |
| 2 | Reject | Shell — add 👎 reaction if unauthorized (runs only on auth failure) |
| 3 | Checkout | Clone the repo |
| 4 | Setup Bun | Install runtime |
| 5 | Preinstall | `indicator.ts` — add 🚀 reaction |
| 6 | Install | `bun install --frozen-lockfile` |
| 7 | Run | `agent.ts` — execute the agent, post reply, commit state |

### 2. Single Runtime Dependency

The `package.json` contains exactly one dependency:

```json
{
  "dependencies": {
    "@mariozechner/pi-coding-agent": "^0.52.5"
  }
}
```

The `pi` coding agent is the entire AI runtime — it handles LLM communication, tool execution (read, edit, bash, grep), session management, and multi-provider support. GMI's orchestrator (`agent.ts`) is a thin shell that invokes `pi` with the right arguments and manages the GitHub-specific lifecycle around it.

This is a deliberate architectural choice: keep the Githubification layer as thin as possible and delegate AI complexity to a battle-tested runtime.

### 3. Issue-Driven Conversation

The same pattern as OpenClaw, proving it's fundamental to Githubification:

```
Issue #7  →  .github-minimum-intelligence/state/issues/7.json  →  .github-minimum-intelligence/state/sessions/<timestamp>.jsonl
```

When a user comments on issue #7 weeks later, the agent loads the linked session file and resumes with full context. No database, no session cookies — just git.

### 4. Git as Memory

Every agent response, every file edit, every session transcript is committed to git. The `agent.ts` orchestrator includes the full commit-and-push logic with retry:

- Stage all changes (`git add -A`)
- Commit only if the index is dirty
- Push with a 10-attempt retry loop, rebasing with `-X theirs` on conflicts
- Backoff delays escalating from 1s to 15s

This gives full auditability (`git log`), rollback (`git revert`), and persistence across sessions.

### 5. Personality Hatching

GMI introduces a pattern not seen in OpenClaw: **agent identity is discovered, not configured.**

The `BOOTSTRAP.md` file is a first-run script that guides a collaborative conversation between the user and the agent to discover the agent's name, nature, vibe, and emoji. The result is written to `AGENTS.md` and `state/user.md`. Every subsequent session begins by reading `AGENTS.md` — the agent remembers who it is.

The GMI reference implementation has already hatched: its agent is named **Spock** 🖖, described as "a rational digital entity instantiated within a CI runner" with a "disciplined, analytical, and precise" vibe.

This pattern makes each Githubified instance feel unique while sharing identical infrastructure.

### 6. Authorization Without Sentinel Files

OpenClaw uses a sentinel file (`GITOPENCLAW-ENABLED.md`) as a fail-closed guard — delete the file and everything stops. GMI takes a different approach: authorization is a **workflow step**, not a file check.

The workflow's `Authorize` step queries the GitHub API for the actor's permission level and only proceeds for `admin`, `maintain`, or `write` roles. If authorization fails, the `Reject` step adds a 👎 reaction and the workflow terminates.

This is simpler (no file to manage) but trades the ability to disable the agent by deleting a file for reliance on GitHub's permission model. Both approaches are valid; the choice depends on whether you want a repo-level kill switch or GitHub-level access control.

### 7. Multi-Provider LLM Support

GMI supports eight LLM providers out of the box through a single configuration file (`.pi/settings.json`):

| Provider | Model Examples |
|----------|------|
| OpenAI | `gpt-5.3-codex`, `gpt-5.3-codex-spark` |
| Anthropic | `claude-sonnet-4-20250514` |
| Google Gemini | `gemini-2.5-pro`, `gemini-2.5-flash` |
| xAI | `grok-3`, `grok-3-mini` |
| DeepSeek | `deepseek/deepseek-r1` (via OpenRouter) |
| Mistral | `mistral-large-latest` |
| Groq | `deepseek-r1-distill-llama-70b` |
| OpenRouter | Any model on openrouter.ai |

The workflow template passes all provider API keys as environment variables, so switching providers requires only a config change and the corresponding secret — no code modifications.

### 8. Modular Skill System

Agent capabilities are packaged as self-contained Markdown files in `.pi/skills/`. The reference implementation ships with two skills:

- **memory** — long-term memory using an append-only `memory.log` file with grep-based retrieval
- **skill-creator** — a meta-skill that creates new skills

Skills are composable and user-extensible. This means the agent's capabilities can grow without modifying the core lifecycle scripts.

### 9. Self-Bootstrapping Installer

GMI provides three installation paths:

1. **Curl-pipe-bash** — `curl -fsSL .../setup.sh | bash` downloads the folder, resets state and config, runs the installer
2. **Manual copy** — Copy the folder, run `bun .github-minimum-intelligence/install/MINIMUM-INTELLIGENCE-INSTALLER.ts`
3. **GitHub App** — Register via `app-manifest.json`, install on repos, the agent runs under its own bot identity

The `MINIMUM-INTELLIGENCE-INSTALLER.ts` creates `.github/workflows/` and `.github/ISSUE_TEMPLATE/`, copies the workflow and issue templates, initializes the identity file, installs dependencies, and prints next steps. It is idempotent — it skips files that already exist.

The `setup.sh` script adds an important detail: it **removes the `state/` directory** and **resets `AGENTS.md` and `settings.json` to defaults** after downloading. This ensures new installations don't inherit the source repo's agent identity or conversation history.

### 10. Concurrency Resilience

The same challenge as OpenClaw, with the same solution: the workflow uses a per-issue concurrency group (`github-minimum-intelligence-${{ github.repository }}-issue-${{ github.event.issue.number }}`) with `cancel-in-progress: false`, so multiple issues can trigger the agent simultaneously without dropping events. Git push conflicts are handled by the 10-attempt retry loop in `agent.ts`.

### 11. Comprehensive Documentation as Architecture

GMI ships with a documentation corpus that goes far beyond a README:

- **Foundational questions** — Six philosophical documents (What? Who? When? Where? How? How Much?) that define the project's architectural thesis
- **DEFCON readiness levels** — Five operational states (DEFCON 1–5) that constrain agent behavior, from "all operations suspended" to "standard operations"
- **Security assessment** — Threat model, vulnerability analysis, access control review
- **Incident response plan** — Step-by-step containment and recovery procedures
- **The Four Laws of AI** — Governing laws for agent behavior
- **Capabilities audit** — Evidence-based analysis of what the agent can and cannot access

This documentation is not supplementary — it's structural. The DEFCON system provides a governance framework that other Githubified repos can adopt. The foundational questions provide a template for reasoning about any Githubification project.

---

## What GMI Teaches About the Spectrum of Type 1

The OpenClaw lesson demonstrated that Githubification can wrap a complex, existing AI agent without modifying it. GMI demonstrates the other end of the spectrum: **what happens when the agent is designed for GitHub from day one.**

| Dimension | OpenClaw (wrapped) | GMI (native) |
|-----------|-------------------|--------------|
| **Agent origin** | External project, ported to GitHub | Built for GitHub from scratch |
| **Githubification layer** | Separate folder alongside agent source | The folder IS the agent |
| **Lifecycle complexity** | 5-step pipeline with sentinel guard | 2-file lifecycle with workflow-level auth |
| **Runtime dependencies** | 30+ tools, vector memory, browser automation | 1 dependency (`pi-coding-agent`) |
| **Security model** | Sentinel file (fail-closed by file deletion) | Workflow authorization step (fail-closed by permission check) |
| **Agent identity** | Configured in `AGENTS.md` | Discovered through hatching conversation |
| **Source code relationship** | Agent source is read-only context | No separate source — folder is the product |
| **Extensibility** | Through the wrapped agent's plugin system | Through `.pi/skills/` Markdown modules |
| **Installation** | Installer workflow creates PR | Installer script + curl-pipe-bash + GitHub App |

Both approaches produce the same result: **an AI agent that runs on GitHub Actions, converses through Issues, persists through Git, and authenticates through Secrets.** The primitives are identical. The path to get there differs.

---

## Lessons for Any Type 1 Githubification

1. **Native beats wrapped for simplicity.** When the agent is designed for GitHub from the start, the orchestration layer can be dramatically thinner. GMI's entire lifecycle is two TypeScript files and a workflow. If you're building a new agent, build it Githubified from day one.

2. **Wrapped beats native for adoption.** When the agent already exists and has a user base, wrapping it (like OpenClaw) preserves the upstream project and avoids forking. If you're Githubifying an existing agent, don't rewrite it — wrap it.

3. **The four primitives are universal.** Both GMI and OpenClaw map to the same four GitHub primitives: Actions (compute), Git (memory), Issues (UI), Secrets (credentials). This mapping is the invariant of Githubification, regardless of whether the agent is wrapped or native.

4. **Minimize runtime dependencies.** GMI's single-dependency approach (`pi-coding-agent`) proves that a production AI agent can run with a minimal footprint. Fewer dependencies mean faster installs, smaller attack surface, and simpler debugging.

5. **Authorization can live in the workflow.** Not every repo needs a sentinel file. GMI's workflow-level authorization step is simpler and leverages GitHub's existing permission model. Choose the approach that matches your threat model.

6. **Make identity a first-class concern.** GMI's hatching system turns agent identity from a configuration file into a collaborative discovery process. This makes each installation feel personal and encourages users to think about what they want from their agent.

7. **Document the philosophy, not just the code.** GMI's foundational questions, DEFCON levels, and Four Laws provide a governance framework that any Githubified repo can adopt. Documentation that explains _why_ decisions were made is more valuable than documentation that only explains _how_ the code works.

8. **Skills as Markdown.** GMI's skill system uses plain Markdown files, not code plugins. This makes capabilities readable, composable, and user-editable without touching TypeScript. When extending a Githubified agent, consider whether the extension can be expressed as a document rather than a code change.

9. **Reset state on distribution.** GMI's `setup.sh` strips the `state/` directory and resets identity files before installing into a new repo. This is critical: a distributed Githubification template must not carry the source repo's conversation history or agent personality into new installations.

10. **The pattern scales down as well as up.** OpenClaw proved that Githubification works for a 30-tool AI platform. GMI proves it works for a minimal, single-dependency agent. The Githubification pattern is invariant to agent complexity — it works at every scale.

---

## Summary

`japer-technology/github-minimum-intelligence` demonstrates that a full AI agent can be born as a single folder in a GitHub repository, with no external runtime, no separate codebase to wrap, and no infrastructure beyond GitHub itself. The agent was designed for GitHub from the start, and the result is a dramatically simpler architecture that still delivers the same capabilities: persistent conversations, full auditability, multi-provider LLM support, and zero-infrastructure deployment.

Where OpenClaw teaches us how to Githubify an existing agent, GMI teaches us what happens when you **start with Githubification as the premise**. The folder is the agent. The repo is the mind. GitHub is the infrastructure.

This is what Type 1 Githubification looks like when the agent is native: **there is no separation between the agent and its Githubification layer, because they were always the same thing.**
