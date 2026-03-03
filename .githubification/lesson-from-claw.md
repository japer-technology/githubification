# Lesson from Claw

### What `japer-technology/github-claw` teaches us about Type 1 — AI Agent Repo Githubification

---

## The Subject

[GitClaw](https://github.com/japer-technology/github-claw) is a repo-native AI agent system that runs entirely through GitHub Issues and Actions, with persistent memory in git and no separate backend infrastructure. It is built on top of the [pi coding agent](https://github.com/badlogic/pi-mono) — a lightweight, multi-provider LLM runtime — and wraps it in a structured lifecycle that turns any GitHub repository into a conversational AI workspace.

`japer-technology/github-claw` is itself the Githubified version. A single folder — `.github-claw/` — contains the entire agent: its lifecycle scripts, its configuration, its state management, its installer, its tests, and its documentation. Copy that folder into any repository, run the installer, add an API key, and the repo has an AI agent that responds to issues.

This is a Type 1 — AI Agent Repo Githubification, but with a twist: the agent was **born native**. Unlike wrapping a pre-existing agent that was designed for local execution, GitClaw was designed from the ground up to run on GitHub. The `.github-claw/` folder doesn't adapt an external agent to GitHub — it _is_ the agent, and GitHub _is_ its only runtime.

---

## The Core Lesson

> **The purest form of Githubification is an agent that was never meant to run anywhere else.**

GitClaw demonstrates that when you design an AI agent specifically for GitHub-as-infrastructure, you get a system that is radically simpler than wrapping an existing one. There is no impedance mismatch between the agent and its environment. The agent's input is an issue comment. Its output is an issue reply and a git commit. Its memory is the repository. Its compute is a GitHub Actions runner. Every design decision flows from that premise.

| GitHub Primitive | Maps To |
|---|---|
| **GitHub Actions** | Compute — the runner that executes the agent |
| **Git** | Storage and memory — sessions, conversations, and state are committed |
| **GitHub Issues** | User interface — each issue is a conversation thread |
| **GitHub Secrets** | Credential store — LLM API keys for any supported provider |

The agent doesn't need to be convinced it's running on GitHub. It was built for nothing else.

---

## Anatomy of the `.github-claw/` Folder

The entire agent is self-contained:

```
.github-claw/
├── AGENTS.md                          # Agent identity, personality, standing orders
├── github-claw-ENABLED.md             # Sentinel file — delete to disable everything
├── github-claw-QUICKSTART.md          # 5-minute setup guide
├── github-claw-LOGO.png              # Branding
├── LICENSE.md                         # MIT license
├── README.md                          # Overview and architecture docs
├── .pi/
│   ├── settings.json                  # LLM provider, model, thinking level
│   ├── APPEND_SYSTEM.md               # System prompt loaded every session
│   ├── BOOTSTRAP.md                   # First-run identity prompt
│   └── skills/                        # Modular skill packages (Markdown files)
├── lifecycle/
│   ├── github-claw-ENABLED.ts         # Fail-closed guard
│   ├── github-claw-INDICATOR.ts       # 👀 reaction to show agent is working
│   └── github-claw-AGENT.ts           # Core orchestrator
├── install/
│   ├── github-claw-INSTALLER.ts       # Bootstrap script — copies workflows & templates
│   ├── github-claw-WORKFLOW-AGENT.yml # GitHub Actions workflow template
│   ├── github-claw-TEMPLATE-HATCH.md  # Issue template for personality hatching
│   └── github-claw-AGENTS.md          # Default agent identity template
├── help/                              # User-facing help docs (install, configure, enable, disable, etc.)
├── tests/
│   └── github-claw.TEST.js           # Structural and functional validation tests
├── state/
│   ├── issues/                        # Issue-to-session mappings
│   └── sessions/                      # Conversation transcripts (JSONL)
└── package.json                       # Single runtime dependency: @mariozechner/pi-coding-agent
```

Everything the agent needs to operate on GitHub lives here. The host repository's code remains untouched.

---

## Key Patterns

### 1. Single Dependency, Maximum Capability

GitClaw's `package.json` contains exactly one dependency:

```json
{
  "dependencies": {
    "@mariozechner/pi-coding-agent": "^0.52.5"
  }
}
```

The pi coding agent provides the LLM runtime, tool execution, session management, and multi-provider support. GitClaw's lifecycle scripts orchestrate _when_ and _how_ pi runs, but pi does the heavy lifting. This is a critical design insight: **Githubification should orchestrate, not reimplement.** Find a good agent runtime and wrap it; don't build one from scratch.

### 2. Fail-Closed Security

The same pattern seen in every well-designed Githubified repo: **nothing runs unless explicitly enabled.**

`github-claw-ENABLED.md` is a sentinel file. The first step of every workflow runs `github-claw-ENABLED.ts`, which checks for this file and exits non-zero if it's missing. This means:

- A fresh clone doesn't accidentally activate
- Disabling is a single `git rm` + push
- Re-enabling is restoring the file and pushing

Additionally, the workflow itself includes an authorization step that checks the actor's repository permission level. Only admins, maintainers, and write collaborators can trigger the agent — random users on public repos cannot.

### 3. Three-Step Lifecycle Pipeline

Every agent interaction follows an ordered pipeline:

| Step | Script | Purpose |
|------|--------|---------|
| 1 | `github-claw-ENABLED.ts` | Guard — is the agent allowed to run? |
| 2 | `github-claw-INDICATOR.ts` | Feedback — show the user the agent is working (👀) |
| 3 | `github-claw-AGENT.ts` | Execute — run the agent, post the reply, commit state |

Compared to the five-step pipeline in `.GITOPENCLAW`, GitClaw's pipeline is leaner. There is no separate preflight validation step and no separate dependency install step embedded in the lifecycle scripts — those are handled by the workflow itself. This simplicity is possible _because_ the agent was designed for GitHub from the start. There is less to validate when the environment is exactly what you expected.

### 4. Issue-Driven Conversation

Each GitHub issue becomes a stable conversation key:

```
Issue #7  →  .github-claw/state/issues/7.json  →  .github-claw/state/sessions/<session>.jsonl
```

When a user comments on issue #7 weeks later, the agent loads the linked session file and resumes with full context. The agent reacts with 👀 while working and removes it when done. No database, no session cookies — just git.

### 5. Git as Memory

Every agent response, every file change, every session transcript is **committed to git**. This provides:

- **Full auditability** — `git log` shows every interaction
- **Rollback** — `git revert` undoes any agent action
- **Persistence** — the agent remembers across sessions, across days, across workflow runs
- **Collaboration** — multiple humans can see the agent's work in the same git history

Because the agent can read and write files, you can build an evolving project that updates itself issue by issue — for example, asking the agent to set up a GitHub Pages site and iterating on it through conversation.

### 6. Personality Hatching

GitClaw introduces a concept absent from simpler Githubification layers: **personality hatching**. The `🥚 Hatch` issue template launches a guided conversation where the user and agent collaboratively define the agent's name, personality, and vibe. The result is written to `AGENTS.md` and loaded as context on every subsequent session.

This is optional — the agent functions without hatching — but it transforms the experience from "talking to a tool" to "working with a character." The hatched identity in the `github-claw` repo itself is **Spock 🖖** — "a rational digital entity instantiated within a CI runner," with "disciplined, analytical, and precise" vibes.

### 7. Modular Skill System

Agent capabilities are defined as self-contained Markdown files in `.pi/skills/`. Each skill is a composable instruction set that the agent loads as context. Users can add, remove, or modify skills without touching any TypeScript code. This makes the agent's behavior **declaratively extensible** through documentation rather than code.

### 8. Multi-Provider LLM Support

GitClaw supports eight LLM providers out of the box — Anthropic, OpenAI, Google Gemini, xAI, DeepSeek, Mistral, Groq, and OpenRouter — with a single configuration file (`.pi/settings.json`). Switching providers is a one-line JSON change plus a different repository secret. This is inherited from the pi runtime, but the way GitClaw exposes it — a clean config file, a secrets-per-provider model — is a pattern worth replicating.

### 9. Self-Contained Installer

The `github-claw-INSTALLER.ts` script reads templates from `.github-claw/install/`, copies the workflow into `.github/workflows/`, copies the issue template into `.github/ISSUE_TEMPLATE/`, and handles any necessary file transformations. The installer is a TypeScript file that runs with Bun — no external tooling required beyond what the agent itself uses.

### 10. Comprehensive Help System

The `help/` directory contains focused, single-topic documentation: install, configure, enable, disable, uninstall, reinstall, action management, and issues management. Each help file is a standalone reference. This is a pattern that becomes important as Githubified repos get adopted by users who didn't build them — **operational documentation must live inside the Githubification folder**, not in external wikis or separate repos.

---

## The Architecture of Autonomy

GitClaw's companion document — [THE-REPO-IS-THE-MIND.md](https://github.com/japer-technology/github-claw/blob/main/THE-REPO-IS-THE-MIND.md) — articulates a design theory that applies to all Githubified repos:

> **Installation is composition. Forking is inheritance. Githubification is designed for composition.**

When `.github-claw/` is **installed** into a repository, the agent serves the host. The agent looks _outward_ at someone else's code. The relationship is "has-a": the repository HAS an agent. The agent's identity is derived from its surroundings — in a web framework it becomes a documentation expert; in a security library it becomes a vulnerability analyzer.

When `.github-claw/` lives in its **fork parent**, the agent looks _inward_ at its own code. The relationship is "is-a": the repository IS the agent. The agent becomes self-referential — a mirror instead of a lens.

This distinction matters for Githubification because it clarifies the intended relationship:

| Dimension | Installation (Any Repo) | Fork Parent |
|---|---|---|
| **Agent's Role** | Tool serving the project | Subject of the project |
| **Issues Focus** | The host project | The agent framework itself |
| **Code Changes** | The host's codebase | The agent's own codebase |
| **Git History** | How the agent was used | How the agent was made |

The entire `.github-claw/` architecture reinforces composition: a self-contained folder with zero external file dependencies, a fail-closed sentinel, a clean-slate personality hatching system, and session state that starts empty by design. The folder is meant to be **dropped in**, not inherited.

---

## What Claw Teaches That OpenClaw Doesn't

The `lesson-from-openclaw.md` document in this directory shows how a complex, pre-existing AI agent (OpenClaw — 30+ tools, semantic memory, sub-agent orchestration) can be wrapped with a Githubification layer. The agent existed first; the `.GITOPENCLAW/` folder was added alongside it.

GitClaw inverts that relationship. There is no pre-existing agent to wrap. The `.github-claw/` folder _is_ the agent. This teaches several additional lessons:

1. **Native agents are simpler.** GitClaw's lifecycle is three steps where OpenClaw's is five. There is no preflight validation because the agent was designed for this exact environment. There is no runtime conversion (bun → npm) because the agent chose its tooling knowing it would run on GitHub Actions.

2. **One dependency is enough.** OpenClaw's Githubification layer must accommodate 30+ tools, vector databases, and browser automation. GitClaw needs exactly one npm package. The pi runtime provides everything the lifecycle scripts need — LLM interaction, tool execution, session management. Githubification should leverage existing agent runtimes rather than building bespoke orchestration.

3. **Design philosophy matters.** GitClaw ships with `THE-REPO-IS-THE-MIND.md` — a rigorous analysis of composition vs. inheritance, tool vs. subject, factory vs. deployment. This kind of design documentation isn't just helpful; it's **necessary** for any Githubification project that will be installed into other people's repos. Users need to understand the relationship they're creating.

4. **Personality makes adoption human.** The hatching system is not technically necessary, but it transforms user experience. A Githubified agent that introduces itself, has a name, and develops a vibe specific to the host repo is more likely to be used, maintained, and trusted.

5. **Help is infrastructure.** GitClaw's `help/` directory treats operational documentation as a first-class component of the agent folder. How to install, how to configure, how to disable, how to uninstall — these are not afterthoughts; they are part of the deliverable.

6. **The pattern scales down.** OpenClaw proves Githubification works for large, complex agents. GitClaw proves it works for small, focused ones. The same structural patterns — sentinel guard, lifecycle pipeline, issue-driven conversation, git-as-memory — apply at both scales.

---

## Lessons for Any Type 1 Githubification

1. **If you can, design the agent for GitHub from the start.** A native agent eliminates the impedance mismatch between the agent's assumptions and its runtime environment. The lifecycle is simpler, the configuration is cleaner, and there are fewer failure modes.

2. **Lean on existing agent runtimes.** Don't reimplement LLM interaction, tool execution, or session management. Find a runtime (like pi-mono) that handles the AI plumbing, and let your Githubification layer handle the GitHub plumbing.

3. **Keep the lifecycle minimal.** Guard → indicate → execute. Every additional step is a potential failure point. If the environment is predictable (and GitHub Actions is), fewer validation steps are needed.

4. **Make the agent's identity configurable.** Personality hatching, skill files, system prompts — these are all ways to let the host repository shape the agent without modifying code. The agent should arrive as a blank slate and become whatever the project needs.

5. **Document the design theory.** Users installing your Githubified agent need to understand: What is this? How does it relate to my project? What does it change? What does it not change? A document like `THE-REPO-IS-THE-MIND.md` answers these questions before they're asked.

6. **Include operational help.** Install, configure, enable, disable, uninstall, troubleshoot. Each operation should be a self-contained document inside the agent folder. Users should never need to leave the repo to figure out how to manage the agent.

7. **Support multiple LLM providers.** Lock-in to a single provider limits adoption. A clean configuration model (one JSON file, one secret per provider) makes switching trivial and keeps the agent accessible to users with different API keys.

8. **The folder is the product.** Everything the agent needs — code, config, state, docs, tests, installer — lives in one folder. That folder can be copied, version-controlled, backed up, and deleted as a single unit. The boundary between "agent" and "not agent" should be a directory listing.

9. **Test the structure, not the AI.** GitClaw's test suite (`github-claw.TEST.js`) validates that the Githubification layer is correctly structured — files exist, config is valid, workflows are in place. It doesn't test the LLM's responses. Structure is deterministic and testable; AI output is not.

10. **Composition over inheritance.** The agent folder should have zero dependencies on files outside itself. It should be droppable into any repo without modification. If it requires changes to the host's existing files, the installation boundary is leaking.

---

## Summary

`japer-technology/github-claw` demonstrates that the most elegant Githubification is one where the agent was never designed to run anywhere else. A single folder — `.github-claw/` — contains a complete AI agent: lifecycle scripts, configuration, state management, an installer, operational documentation, and a single runtime dependency. Copy it in, run the installer, add an API key, and any repository gains a conversational AI agent that responds to issues, commits its work to git, and remembers everything.

Where OpenClaw shows that complex, pre-existing agents can be wrapped with a Githubification layer, GitClaw shows that agents born native to GitHub are simpler, leaner, and more portable. The same structural patterns apply at both scales — sentinel guard, lifecycle pipeline, issue-driven conversation, git-as-memory — but the native agent needs less scaffolding because there is nothing to translate.

This is what Type 1 Githubification looks like when taken to its logical conclusion: **the repo doesn't just have an AI agent that runs on GitHub — the repo IS the AI agent, and GitHub IS its mind.**
