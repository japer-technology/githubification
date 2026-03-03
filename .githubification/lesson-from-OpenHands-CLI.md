# Lesson from OpenHands CLI

### What `japer-technology/github-OpenHands-CLI` teaches us about Type 1 — AI Agent Repo Githubification

---

## The Subject

[OpenHands CLI](https://github.com/OpenHands/OpenHands-CLI) is a standalone terminal interface for the OpenHands AI coding agent. It provides a Textual-based TUI, IDE integration (via Agent-Computer Protocol), headless mode for CI/CD pipelines, a browser-served web view, and cloud sandbox execution — all unified under a single `openhands` command. It is built in Python 3.12 with the OpenHands Software Agent SDK, packaged as both a PyPI package (`uv tool install openhands`) and a standalone binary via PyInstaller, and licensed under MIT. The agent it exposes can read and write code, execute shell commands, browse the web, and interact with Model Context Protocol (MCP) servers — making it a general-purpose AI coding agent with multiple delivery surfaces.

`japer-technology/github-OpenHands-CLI` is the Githubification fork. Like the AutoGPT case study, **no Githubification folder has been added yet** — there is no `.issue-intelligence/`, no lifecycle scripts, no sentinel file, no issue-driven conversational agent. What the fork preserves from upstream is something more distinctive: a repository where **the AI agent is already running on GitHub Actions, operating on its own codebase**. OpenHands CLI uses OpenHands itself for PR reviews, code quality analysis, automated issue labeling, and code fixes. All of these run through GitHub Actions workflows that invoke the CLI in headless mode. The agent is not yet running _for users_ through Issues, but it is already running _for developers_ through the CI/CD pipeline.

This is a Type 1 — AI Agent Repo that reveals a phase of Githubification not seen in the AutoGPT case study: **self-hosted agent automation**, where the AI agent in the repository is deployed on GitHub Actions to maintain the repository itself. The agent has a GitHub-native execution path (headless mode + Actions workflows) but that path is pointed inward at development, not outward at users.

---

## The Core Lesson

> **When an AI agent already runs in headless mode on GitHub Actions — maintaining its own repository through PR reviews, code quality fixes, and issue triage — the infrastructure for Githubification already exists. The remaining step is redirecting that execution path from developer-facing automation to user-facing conversation.**

The previous case studies treat GitHub Actions as blank infrastructure that must be configured from scratch. GitClaw builds a lifecycle pipeline. Agent Zero requires substitution because its runtime conflicts with Actions. AutoGPT is still in the preparation phase, making the codebase legible. OpenHands CLI occupies a different position: the agent already executes on GitHub Actions runners, already reads the repository, already produces commits and PR comments, and already uses GitHub Secrets for LLM API keys. Every Githubification primitive is not just present but **actively in use** — just not for the purpose Githubification envisions.

| GitHub Primitive | Current Use | Githubification Use |
|---|---|---|
| **GitHub Actions** | Compute — runs the agent in headless mode for PR reviews, code quality analysis, issue labeling, and automated fixes | Compute — would run the agent in headless mode for user conversations via Issues |
| **Git** | Storage — the codebase the agent reads and modifies through PRs | Storage — would additionally store conversation sessions and agent memory |
| **GitHub Issues** | Human interface — used for traditional bug/feature tracking, auto-labeled by the agent | User interface — would become the conversational surface for interacting with the agent |
| **GitHub Secrets** | Credential store — `LLM_API_KEY`, `LLM_BASE_URL`, `ALLHANDS_BOT_GITHUB_PAT` | Credential store — same secrets, same purpose, now serving user-facing conversations |

The four primitives are mapped and operational. The gap is directional, not structural.

---

## Anatomy of the Repository

The repository has four distinct layers:

### Layer 1: The CLI Application (`openhands_cli/`)

The core product — a multi-surface interface for the OpenHands agent:

```
openhands_cli/
├── __init__.py                    # Package metadata (version 1.13.0)
├── entrypoint.py                  # Main CLI entry point (typer-based)
├── tui/                           # Textual TUI — the primary interactive interface
│   └── core/
│       ├── state.py               # ConversationContainer — reactive state holder
│       ├── conversation_manager.py # Message router — delegates to controllers
│       └── ...                    # Controllers, widgets, modals, settings
├── acp_impl/                      # Agent-Computer Protocol implementation (IDE integration)
├── auth/                          # OAuth device flow, token storage, HTTP client
├── cloud/                         # OpenHands Cloud integration
├── conversations/                 # Conversation CRUD, local storage
├── mcp/                           # Model Context Protocol server management
├── shared/                        # Extracted shared utilities (slash commands, etc.)
├── stores/                        # State persistence layer
├── gui_launcher.py                # Docker-based GUI server launcher
├── setup.py                       # First-run configuration wizard
├── utils.py                       # Shared utilities
├── locations.py                   # File path constants
├── theme.py                       # TUI theming
├── terminal_compat.py             # Terminal compatibility helpers
└── version_check.py               # PyPI version check
```

The CLI supports five running modes: interactive TUI (`openhands`), IDE integration (`openhands acp`), headless (`openhands --headless -t "task"`), browser-served web (`openhands web`), and GUI server (`openhands serve`). The headless mode is the critical one for Githubification — it accepts a task string or file, runs the agent without interactive UI, and exits with a result. This is the mode already used in GitHub Actions workflows.

### Layer 2: Agent Infrastructure

The agent runtime is external to this repository — provided by the `openhands-sdk` and `openhands-tools` packages — but the CLI configures and invokes it:

```
pyproject.toml                     # Dependencies: openhands-sdk==1.11.5, openhands-tools==1.11.5
~/.openhands/                      # Runtime configuration (created on first run)
├── agent_settings.json            # Agent settings, condenser config
├── cli_config.json                # CLI/TUI preferences
└── mcp.json                       # MCP server configuration
```

The agent supports multiple LLM providers (configured via `LLM_MODEL`, `LLM_API_KEY`, `LLM_BASE_URL`), MCP server extensions, conversation condensation for long contexts, and confirmation policies ranging from manual approval to full autonomy (`--always-approve` / `--yolo`). The `--override-with-envs` flag allows environment variables to override persisted settings — the exact mechanism used in GitHub Actions workflows.

### Layer 3: Self-Hosted Agent Automation

The distinctive feature — GitHub Actions workflows that use the OpenHands CLI to maintain the repository:

```
.github/
├── workflows/
│   ├── pr-review-by-openhands.yml     # OpenHands reviews every PR (Claude-based)
│   ├── pr-review-evaluation.yml       # Evaluates how well PR review comments were addressed
│   ├── good-first-issue-labeler.yml   # Weekly: OpenHands labels new issues as good-first-issue
│   ├── tests.yml                      # Unit tests, snapshot tests, coverage reporting
│   ├── lint.yml                       # Pre-commit checks (ruff, pycodestyle, pyright, yamlfmt)
│   ├── bump-version.yml               # Automated version bumping with snapshot regeneration
│   ├── bump-agent-sdk-version.yml     # Automated SDK dependency updates
│   ├── cli-build-binary-and-optionally-release.yml  # PyInstaller binary builds
│   ├── pypi-release.yml               # PyPI package publishing
│   ├── check-package-versions.yml     # Dependency version verification
│   ├── update-install-website.yml     # Post-release website update
│   ├── update-pr-description.yml      # Auto-update PR descriptions
│   ├── stale.yml                      # Stale issue/PR management
│   └── type-checking-report.yml       # Pyright type checking analysis
├── prompts/
│   ├── auto-fix-low-hanging.md        # Prompt for automated code quality fixes
│   ├── code-quality-analysis.md       # Prompt for comprehensive code quality analysis
│   └── good-first-issue-labeler.md    # Prompt for issue labeling
└── scripts/
    └── update_pr_description.sh       # PR description automation
```

Three workflows are agent-driven:

1. **PR Review** (`pr-review-by-openhands.yml`): On every non-draft PR from a non-first-time contributor, OpenHands runs as a reviewer using Claude, analyzing the diff and posting review comments. The review style is configurable ("roasted" or "standard"). A companion evaluation workflow (`pr-review-evaluation.yml`) assesses review quality after the PR is closed.

2. **Good First Issue Labeling** (`good-first-issue-labeler.yml`): Weekly on a schedule, the workflow runs OpenHands in headless mode with a prompt file that instructs it to query the GitHub API, analyze recent issues, and apply the `good first issue` label to well-scoped, low-risk issues. The agent uses the GitHub token to make API calls directly.

3. **Code Quality Analysis** (via `.github/prompts/`): The `code-quality-analysis.md` prompt instructs OpenHands to analyze the codebase for type checking issues, state management problems, separation of concerns violations, and ACP/TUI code duplication. The `auto-fix-low-hanging.md` prompt then instructs it to automatically fix the trivial findings, create branches, run tests, and open focused PRs.

### Layer 4: Developer Tooling and Quality Infrastructure

The toolchain that ensures the agent's contributions (and human contributions) meet quality standards:

```
Makefile                           # build, install, test, lint, format, run targets
.pre-commit-config.yaml            # ruff format, ruff check, pycodestyle, pyright, yamlfmt
.openhands/
├── hooks.json                     # OpenHands lifecycle hooks
└── hooks/
    └── on_stop.sh                 # Pre-commit gate — blocks agent stop if linting fails
AGENTS.md                          # 13KB structured onboarding for AI agents
build.py                           # PyInstaller build script with automated testing
build.sh                           # Build shell wrapper
openhands-cli.spec                 # PyInstaller spec file
```

The `.openhands/hooks.json` deserves special attention. It defines a `stop` hook that runs `.openhands/hooks/on_stop.sh` before the OpenHands agent can complete a task. The hook runs `pre-commit run --all-files` and returns a JSON decision: `{"decision": "allow"}` if all checks pass, or `{"decision": "deny", "reason": "pre-commit failed"}` if they fail. This is a **quality gate for the agent itself** — the AI cannot declare its work done unless the code passes linting, formatting, and type checking. This pattern enforces code quality at the agent level, not just the CI level.

---

## Key Patterns

### 1. The Agent Is Already on GitHub Actions

This is the defining pattern. In every other case study, deploying the agent to GitHub Actions is the deliverable — the thing Githubification creates. In OpenHands CLI, the agent is already there:

```yaml
# From good-first-issue-labeler.yml
- name: Run OpenHands labeler
  env:
    LLM_API_KEY: ${{ secrets.LLM_API_KEY }}
    LLM_BASE_URL: https://llm-proxy.app.all-hands.dev
    LLM_MODEL: litellm_proxy/gpt-5.2
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    uv run openhands --headless --always-approve --override-with-envs \
      --file .github/prompts/good-first-issue-labeler.md
```

This workflow already demonstrates the complete Githubification execution model: a GitHub Actions runner installs the agent, configures it with secrets, passes it a task via a prompt file, and lets it operate autonomously (`--always-approve`). The agent reads the repository, calls the GitHub API, and makes changes. The only difference from a Githubification workflow is the trigger (scheduled cron vs. issue comment) and the audience (maintainers vs. users).

### 2. Prompt Files as Task Specifications

The `.github/prompts/` directory introduces a pattern not seen in other case studies: **structured prompt files that define agent tasks as reusable Markdown documents**. Instead of embedding the agent's instructions in the workflow YAML, the task is defined in a separate file:

| Prompt File | Size | Purpose |
|---|---|---|
| `auto-fix-low-hanging.md` | 3.2KB | Instructions for fixing trivial code quality issues, including branch creation, testing, and PR submission |
| `code-quality-analysis.md` | 3.8KB | Instructions for comprehensive code analysis: type checking, state management, separation of concerns, duplication |
| `good-first-issue-labeler.md` | 1.3KB | Instructions for identifying and labeling good-first-issue candidates |

Each prompt is a self-contained task specification: context, rules, expected behavior, and output format. The workflow invokes the agent with `--file .github/prompts/<prompt>.md`, making the prompt file the unit of agent behavior. This is analogous to GitClaw's skill files (`.pi/skills/`) but operates at a higher level — each prompt defines an entire workflow, not a reusable capability.

For a Githubification layer, this pattern could extend naturally: a `respond-to-issue.md` prompt that instructs the agent to read an issue, understand the question, consult the codebase, and post a reply. The infrastructure for loading and executing prompt files already exists.

### 3. Self-Referential Agent Development

OpenHands CLI is an AI coding agent that uses an AI coding agent to develop itself. The commit history demonstrates this:

- `fix(types): add missing type hints...` — Co-authored-by: `openhands <openhands@all-hands.dev>`
- `refactor(shared): extract parse_slash_command...` — Co-authored-by: `openhands <openhands@all-hands.dev>`
- `fix(acp): lazy-init ACP_CACHE_DIR...` — Co-authored-by: `openhands <openhands@all-hands.dev>`

These are not human-written patches committed under an automated account. They are changes proposed by the OpenHands agent through the code quality analysis workflow, reviewed by human maintainers, and merged. The agent analyzed the codebase (following the `code-quality-analysis.md` prompt), identified improvements, created branches, ran tests, and opened PRs (following the `auto-fix-low-hanging.md` prompt).

This self-referential pattern is unique among the case studies. GitClaw doesn't modify its own code. AutoGPT has AI agents configured to develop the software, but they are external agents (Copilot, Claude, Codex) — not AutoGPT itself. OpenHands CLI is the only case where the agent in the repository is the same agent that develops the repository.

### 4. Agent Quality Gates via Lifecycle Hooks

The `.openhands/hooks.json` and `on_stop.sh` implement a pattern that is conceptually identical to GitClaw's fail-closed guard, but operates at the agent runtime level rather than the workflow level:

```json
{
  "stop": [{
    "matcher": "*",
    "hooks": [{
      "type": "command",
      "command": ".openhands/hooks/on_stop.sh",
      "timeout": 300
    }]
  }]
}
```

When the OpenHands agent attempts to finish a task, the stop hook runs all pre-commit checks. If they fail, the hook returns `{"decision": "deny"}` and the agent must fix the issues before it can stop. This creates a quality feedback loop: the agent proposes changes, the hook validates them, and if they fail, the agent iterates until they pass.

GitClaw achieves a similar effect through its lifecycle pipeline (ENABLED check → INDICATOR → AGENT → response), but the guard operates at the workflow level — if the sentinel file is missing, the workflow exits. OpenHands CLI's guard operates at the agent task level — the agent cannot complete any task without passing quality checks. This is a finer-grained quality gate that applies to every agent interaction, not just the conversational lifecycle.

### 5. Headless Mode as the Githubification Interface

The `--headless` flag is the bridge between "local AI agent" and "GitHub-native AI agent." When invoked headlessly:

```bash
openhands --headless --always-approve --override-with-envs -t "task description"
openhands --headless --always-approve --override-with-envs --file instructions.md
```

The CLI skips the TUI entirely, runs the agent against the task, and exits. Combined with `--always-approve` (autonomous execution) and `--override-with-envs` (configuration from environment variables), this produces an agent invocation that is purely command-line, stateless, and suitable for CI runners. The `--json` flag adds structured output for parsing.

No other case study has this clean a separation between interactive and non-interactive modes. GitClaw has no interactive mode — it is always headless. Agent Zero has no headless mode — it requires a Flask server. AutoGPT has no headless mode — it requires its full platform stack. OpenHands CLI is uniquely positioned because the same binary runs both interactively and headlessly, with the headless path already proven in production on GitHub Actions.

### 6. PR Review as Proto-Githubification

The `pr-review-by-openhands.yml` workflow is the closest thing to Githubification in the repository. It triggers on PR events (opened, ready_for_review, labeled), runs the OpenHands agent, and posts review comments to the PR. The agent reads the diff, analyzes the changes, and provides feedback.

This is structurally similar to a Githubification workflow that triggers on issue events, runs the agent, and posts replies to the issue. The differences are:

| Dimension | PR Review (Current) | Issue Conversation (Githubification) |
|---|---|---|
| **Trigger** | `pull_request` events | `issues` / `issue_comment` events |
| **Input** | PR diff, title, body | Issue body, comment thread |
| **Output** | Review comments on the PR | Reply comments on the issue |
| **Context** | Code changes + repository | Full repository + conversation history |
| **Scope** | Single review per PR event | Multi-turn conversation per issue |

The PR review workflow also includes a companion evaluation workflow (`pr-review-evaluation.yml`) that assesses review quality after the PR is closed. This feedback loop — agent reviews code, then the review itself is evaluated — is a maturity indicator not seen in any other case study.

### 7. Comprehensive AGENTS.md as Agent Context

The `AGENTS.md` is a 13KB structured onboarding document covering:

- **Project structure and module organization** — directory-by-directory explanation
- **Setup, build, and development commands** — exact `make` targets and `uv run` commands
- **Development guidelines** — linting requirements, typing conventions, documentation rules
- **Coding style and naming conventions** — ruff configuration, Python 3.12 patterns
- **Testing guidelines** — unit, snapshot, binary, and integration test strategies
- **Snapshot testing with pytest-textual-snapshot** — complete guide with code examples
- **Commit and PR guidelines** — scope prefixes, commit format, verification flow
- **Security and configuration tips** — API key handling, packaging safety
- **TUI state management architecture** — reactive state, controller pattern, widget hierarchy, data flow

This document serves the same purpose as AutoGPT's `.github/copilot-instructions.md` — making the codebase comprehensible to AI agents from cold start. But whereas AutoGPT needs the document because of its monorepo complexity, OpenHands CLI needs it because the TUI architecture (reactive state, Textual widgets, message-based communication) is unusual enough that an AI agent cannot infer the patterns from code alone.

For Githubification, this document provides ready-made system prompt context. An issue-driven agent that loads `AGENTS.md` as context would understand how to navigate the codebase, run tests, follow coding conventions, and produce contributions that match the project's standards.

### 8. MCP as an Extensibility Layer

The CLI's MCP (Model Context Protocol) support allows users to extend the agent with external tool servers:

```bash
openhands mcp add tavily --transport stdio \
  npx -- -y mcp-remote "https://mcp.tavily.com/mcp/?tavilyApiKey=<key>"
```

MCP servers are configured in `~/.openhands/mcp.json` and provide the agent with additional capabilities (web search, database access, API integration) without modifying the agent's core code. For Githubification, MCP opens a path not available to other case studies: an issue-driven agent that can be extended with project-specific tools. A Githubification workflow could configure MCP servers via environment variables or repository configuration, giving the agent access to project databases, monitoring systems, or domain-specific APIs.

---

## What OpenHands CLI Teaches That the Other Lessons Don't

### 1. The Gap Between Self-Hosted Automation and Githubification Is Small

In every other case study, Githubification requires building substantial new infrastructure: lifecycle scripts, state management, workflow files, sentinel files, installers. OpenHands CLI already has the agent running on Actions, already has prompt-file-based task specification, already has quality gates, and already has the secrets configured. The remaining work to create a Githubification layer is primarily about triggers (issue events instead of PR events or cron schedules) and conversation management (multi-turn state instead of single-shot tasks).

### 2. Headless Mode Is the Universal Githubification Interface

The clean separation between interactive and headless execution means that any CLI agent with a headless mode can be Githubified by the same pattern: a workflow that triggers on issue events, passes the issue body as a task, runs the agent headlessly, and posts the response. OpenHands CLI's `--headless -t "task" --always-approve --override-with-envs` is a template for this pattern. The agent doesn't need to know it's running on GitHub — it just needs a task, an environment, and autonomy.

### 3. Self-Referential Development Is a New Pattern

None of the other case studies feature an agent that develops itself. GitClaw is developed by humans. AutoGPT is developed by external AI agents. OpenHands CLI is developed by OpenHands — the same agent the repository contains. This creates a unique feedback loop: improvements to the agent immediately improve the agent's ability to develop itself. When the code quality analysis workflow finds a type error in the CLI, and the auto-fix workflow creates a PR to fix it, the resulting improvement to the CLI makes the agent more capable for the next analysis.

### 4. Prompt Files Are a Higher-Level Abstraction Than Skills

GitClaw uses skill files — Markdown documents that teach the agent specific capabilities (e.g., "how to search the codebase," "how to write tests"). OpenHands CLI uses prompt files — Markdown documents that define complete tasks with context, rules, and expected behavior. Skills are reusable building blocks that the agent combines during a conversation. Prompt files are complete task specifications that the agent executes end-to-end.

For Githubification, prompt files suggest a pattern where different issue labels or templates trigger different prompt files: a `help-wanted` issue might trigger `respond-to-question.md`, a `bug` issue might trigger `investigate-bug.md`, and a `feature-request` issue might trigger `assess-feature.md`. The workflow routes the issue to the appropriate prompt, and the agent executes the task.

### 5. Agent Quality Gates Are More Effective Than CI Quality Gates

The `.openhands/hooks/on_stop.sh` pattern — blocking the agent from completing until pre-commit passes — catches quality issues before they reach CI. In the traditional Githubification model (GitClaw, etc.), the agent produces output and CI validates it after the fact. In OpenHands CLI, the agent is forced to iterate until its output passes quality checks, producing clean commits from the start. This reduces CI failures, PR churn, and review burden.

### 6. The Agent's Runtime Requirements Are Already GitHub-Compatible

Unlike Agent Zero (which requires Flask, FAISS, Docker) or AutoGPT (which requires PostgreSQL, Redis, RabbitMQ, Supabase), OpenHands CLI's runtime requirements are minimal: Python 3.12, a few pip packages, and an LLM API key. The headless mode adds no infrastructure beyond what a GitHub Actions runner provides. This means Githubification can use the **native strategy** (running the actual agent, not a substitute) — the strongest form of Githubification, where users interact with the real agent, not a lightweight proxy reading the codebase.

---

## Lessons for Any Type 1 Githubification

1. **Check whether the agent already runs on GitHub Actions.** If the project has CI/CD workflows that invoke the agent (for testing, code quality, reviews, or any other purpose), the infrastructure for Githubification already exists. The work is redirecting that execution path from developer automation to user conversation.

2. **Headless mode is the prerequisite for native Githubification.** An agent that can only run interactively cannot run on a CI runner. An agent with a clean headless interface — task in, result out, no interactive prompts — can be Githubified natively. When evaluating a Type 1 repo, check for headless/non-interactive execution modes first.

3. **Use prompt files for task specification.** Instead of embedding agent instructions in workflow YAML or hardcoding behavior, define tasks as separate Markdown files that the agent loads at runtime. This makes agent behavior auditable, version-controlled, and modifiable without changing workflow infrastructure.

4. **Implement agent-level quality gates.** Don't rely solely on CI to catch quality issues in agent-produced code. Use lifecycle hooks or pre-completion checks that force the agent to iterate until its output meets quality standards. This produces cleaner commits and reduces review burden.

5. **Leverage self-hosted automation as proof of concept.** If the agent already reviews PRs, labels issues, or fixes code on GitHub Actions, demonstrate this to stakeholders as evidence that Githubification is feasible. The step from "agent reviews PRs" to "agent responds to issues" is smaller than the step from "no agent on GitHub" to "agent responds to issues."

6. **The native strategy is viable when runtime requirements are light.** If the agent runs with only a language runtime and pip packages — no databases, no persistent services, no Docker orchestration — it can execute directly on GitHub Actions runners. Reserve the substitution strategy for agents with heavy infrastructure requirements.

7. **Design for multi-turn conversation from the start.** The current GitHub Actions workflows are single-shot: trigger, run agent, produce output, exit. Githubification requires multi-turn conversation: issue opened, agent responds, user follows up, agent responds again. The conversation state must persist between workflow runs — either in issue comments (stateless replay) or in committed session files (stateful, like GitClaw).

8. **MCP extensibility makes the agent project-specific.** A generic AI coding agent responds to issues with generic capabilities. An agent configured with MCP servers for the project's domain (database access, monitoring, API integration) responds with project-specific knowledge. Plan for MCP configuration as part of the Githubification setup.

9. **Self-referential development is a maturity signal.** When an AI agent is capable enough to develop its own codebase — finding bugs, proposing fixes, passing its own quality gates — it is more than capable of handling user-facing conversations about that codebase. Self-referential development demonstrates the agent's understanding of the code at a depth that no documentation can substitute for.

10. **The four primitives are already mapped in any modern CI/CD repository.** GitHub Actions is compute, Git is storage, Issues are the UI, and Secrets are the credential store. For repositories that already use all four primitives for development automation, Githubification is not about creating new infrastructure — it is about repurposing existing infrastructure for a new audience.

---

## Summary

`japer-technology/github-OpenHands-CLI` demonstrates what happens when an AI agent repository is already running the agent on GitHub Actions for its own development. OpenHands CLI is a Python-based AI coding agent with a Textual TUI, IDE integration, and a clean headless mode. The repository uses OpenHands itself for PR reviews (via Claude), weekly issue labeling, code quality analysis, and automated code fixes — all through GitHub Actions workflows that invoke the CLI with `--headless --always-approve --override-with-envs` and structured prompt files.

No Githubification folder has been added. But the infrastructure for Githubification is more complete than in any other case study: the agent runs on Actions, secrets are configured, prompt files define tasks, lifecycle hooks enforce quality, the `AGENTS.md` provides comprehensive agent context, and the headless mode produces clean command-line execution on CI runners. The agent's runtime requirements — Python 3.12 and pip packages — are fully compatible with GitHub Actions, making the native strategy viable.

Where GitClaw teaches native design, Agent Zero teaches substitution, AutoGPT teaches preparation, and the consolidation lesson teaches the taxonomy, OpenHands CLI teaches **proximity** — how close a well-automated repository already is to Githubification without knowing it. The agent is running. The primitives are mapped. The quality gates are in place. The remaining work is a workflow trigger on `issues` events, a prompt file for conversation, and a state management strategy for multi-turn interaction. The distance from "AI agent that maintains its own repository" to "AI agent that serves users through Issues" is shorter than any previous case study suggests.
