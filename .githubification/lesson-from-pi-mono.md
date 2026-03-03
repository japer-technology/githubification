# Lesson from Pi Mono

### What `japer-technology/github-pi-mono` teaches us about Type 1 — AI Agent Repo Githubification

---

## The Subject

[Pi Mono](https://github.com/japer-technology/github-pi-mono) is an open-source monorepo containing the tools for building AI agents and managing LLM deployments. Its central package — `@mariozechner/pi-coding-agent` — is an interactive coding agent CLI that handles LLM communication, tool execution (read, edit, bash, grep), session management, multi-provider support, and terminal UI rendering. Around this core sit six supporting packages: a unified multi-provider LLM API (`pi-ai`), an agent runtime with tool calling and state management (`pi-agent-core`), a Slack bot that delegates messages to the coding agent (`pi-mom`), a terminal UI library with differential rendering (`pi-tui`), web components for AI chat interfaces (`pi-web-ui`), and a CLI for managing vLLM deployments on GPU pods (`pi-pods`).

`japer-technology/github-pi-mono` is this repository as it exists today — **not yet Githubified**. There is no issue-triggered workflow, no conversational agent running on GitHub Actions, no Githubification folder. The repository uses GitHub for standard development infrastructure: continuous integration (build → check → test), a contributor gating system (PR Gate + Approve Contributor workflows), tag-triggered binary builds, structured issue templates, and pre-commit hooks. GitHub Actions compiles the TypeScript packages, runs the test suite, and produces cross-platform binaries via Bun — but the coding agent itself does not execute on GitHub in response to issues.

What makes pi-mono uniquely instructive is its position in the Githubification ecosystem: **it is the agent that powers Githubification itself.** GitHub Minimum Intelligence, GitClaw, and every native-strategy Githubified repo studied in this collection depends on `@mariozechner/pi-coding-agent` as its sole runtime dependency. Pi-mono is the foundation layer — the agent engine that other repos wrap, invoke, and configure. Studying it reveals not just how to Githubify an AI agent repo, but how the agent that enables Githubification is itself structured.

This is a **pre-Githubification Type 1 subject**: a full AI agent studied at the moment before a Githubification layer would be applied.

---

## The Core Lesson

> **The agent that enables Githubification for others has not been Githubified itself — and its architecture reveals why the monorepo-as-foundation pattern is the hardest and most valuable Type 1 case.**

Pi-mono is not a single agent. It is a _system_ of packages that compose into an agent. The coding agent depends on the agent runtime, which depends on the AI library, which abstracts over eight LLM providers. The terminal UI, web UI, and Slack bot are alternative frontends to the same underlying agent loop. This layered architecture means Githubification cannot simply wrap "the agent" — it must decide _which layer_ to invoke and _how_ the monorepo's internal package dependencies resolve at runtime on a GitHub Actions runner.

Yet the architecture also provides everything a Githubification layer needs:

| GitHub Primitive | Maps To |
|---|---|
| **GitHub Actions** | Compute — the runner executes the coding agent via `npx tsx packages/coding-agent/src/cli.ts` or downloads a pre-built binary from GitHub Releases |
| **Git** | Storage and memory — the agent already manages sessions and state via the filesystem; git commits provide persistence |
| **GitHub Issues** | User interface — the agent's CLI interface accepts text input and produces text output, directly mappable to issue comments |
| **GitHub Secrets** | Credential store — the agent reads API keys from environment variables (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, etc.) which map directly to GitHub Secrets |

The mapping is clean because pi-mono's design choices align with GitHub's execution model: stateless CLI invocation, environment-variable configuration, filesystem-based state, and text-in/text-out interaction. The agent was not designed for GitHub, but it was designed in a way that makes GitHub a natural host.

---

## Anatomy of the Repository

Pi-mono is a conventional, well-structured TypeScript monorepo — not a Githubification folder, but a complete development ecosystem:

```
github-pi-mono/
├── .WORKFLOW-DESIGN-THEORY.md         # Design document explaining all workflows
├── .github/
│   ├── APPROVED_CONTRIBUTORS          # Allowlist for the contributor gating system
│   ├── APPROVED_CONTRIBUTORS.vacation # Backup during OSS vacation mode
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug.yml                    # Structured bug report form
│   │   ├── contribution.yml           # Contribution proposal (gating entry point)
│   │   └── config.yml                 # Disables blank issues, redirects to Discord
│   ├── github-pi.png                  # Project branding
│   └── workflows/
│       ├── ci.yml                     # CI: build → check → test (push/PR to main)
│       ├── pr-gate.yml                # PR Gate: contributor allowlist enforcement
│       ├── approve-contributor.yml    # Approve Contributor: lgtm → auto-approve
│       └── build-binaries.yml         # Release: tag-triggered cross-platform binaries
├── .gitignore
├── .husky/
│   └── pre-commit                     # Runs `npm run check`, restages formatted files
├── .pi/                               # Pi agent configuration for development
│   ├── extensions/                    # Custom tools: diff, files, prompt-url, redraws, tps
│   ├── git/                           # Git-related config (.gitignore)
│   ├── npm/                           # npm-related config (.gitignore)
│   └── prompts/                       # Reusable prompts: cl.md, is.md, pr.md
├── AGENTS.md                          # Development rules for humans AND AI agents (~9KB)
├── CONTRIBUTING.md                    # Contribution guide with approval gate process
├── LICENSE                            # MIT
├── README.md                          # Package overview and development commands
├── biome.json                         # Biome linter/formatter config (tabs, 120 cols)
├── package.json                       # Workspace root with lockstep versioning scripts
├── package-lock.json                  # Reproducible dependency tree
├── packages/
│   ├── ai/                            # @mariozechner/pi-ai — unified multi-provider LLM API
│   ├── agent/                         # @mariozechner/pi-agent-core — agent runtime + tools
│   ├── coding-agent/                  # @mariozechner/pi-coding-agent — the CLI coding agent
│   ├── mom/                           # @mariozechner/pi-mom — Slack bot frontend
│   ├── tui/                           # @mariozechner/pi-tui — terminal UI library
│   ├── web-ui/                        # @mariozechner/pi-web-ui — web chat components
│   └── pods/                          # @mariozechner/pi-pods — vLLM GPU pod management
├── pi-mono.code-workspace             # VS Code workspace config
├── pi-test.sh                         # Run pi from source (development)
├── test.sh                            # Run tests without API keys (CI-safe)
├── scripts/
│   ├── build-binaries.sh              # Cross-platform binary compilation via Bun
│   ├── cost.ts                        # LLM cost analysis tooling
│   ├── release.mjs                    # Release automation (version bump → publish)
│   ├── session-transcripts.ts         # Session transcript analysis
│   └── sync-versions.js               # Lockstep version synchronization across packages
├── tsconfig.json                      # Root TypeScript config with workspace path mappings
└── tsconfig.base.json                 # Shared TypeScript compiler options
```

The structure is a conventional monorepo, but several design choices are directly relevant to Githubification.

---

## Key Patterns

### 1. The Agent Is a Package, Not a Binary

Pi-mono publishes its coding agent as an npm package (`@mariozechner/pi-coding-agent`). This is the design decision that makes the entire native Githubification strategy possible. When GMI or GitClaw declares a single dependency:

```json
{
  "dependencies": {
    "@mariozechner/pi-coding-agent": "^0.52.5"
  }
}
```

…they get the full agent runtime — LLM communication, tool execution, session management, multi-provider support — in a single `npm install`. The agent is not something you clone and build; it's something you install and invoke. This is why native-strategy Githubification folders can have exactly one runtime dependency.

But pi-mono itself cannot use this shortcut. Inside the monorepo, packages reference each other through TypeScript path mappings, not npm registry installs. Running the agent from source requires `npm install && npm run build` across the full dependency chain: `tui → ai → agent → coding-agent`. A Githubification layer for pi-mono would need to either build from source (slow, ~60s+) or download a pre-built binary from GitHub Releases (fast, already available via `build-binaries.yml`).

### 2. Seven Packages, One Agent

The monorepo contains seven packages, but only one of them is the agent:

| Package | Role | Githubification Relevance |
|---------|------|---------------------------|
| `pi-ai` | Multi-provider LLM API | Abstraction layer — supports OpenAI, Anthropic, Google, xAI, DeepSeek, Mistral, Groq, OpenRouter |
| `pi-agent-core` | Agent runtime with tool calling | The execution engine — manages tool loops, state, and message history |
| `pi-coding-agent` | Interactive coding agent CLI | **The agent itself** — what a Githubification layer would invoke |
| `pi-mom` | Slack bot frontend | An existing channel adapter — proof that the agent supports multiple frontends |
| `pi-tui` | Terminal UI library | Display layer — irrelevant for headless GitHub execution |
| `pi-web-ui` | Web chat components | Display layer — irrelevant for headless GitHub execution |
| `pi-pods` | vLLM GPU pod management | Infrastructure tooling — separate from the agent |

This decomposition is significant. Pi-mom proves that the coding agent already supports being invoked from a non-terminal context — Slack messages go in, agent responses come out. A GitHub Issues adapter would follow the same pattern: issue comments go in, agent responses come out. The channel abstraction already exists.

### 3. AGENTS.md as Agent Constitution

Pi-mono's `AGENTS.md` is a ~9KB document that serves as the development constitution for both human and AI contributors. It covers:

- **First-message protocol** — what to do when no task is given
- **Code quality rules** — no `any` types, no inline imports, no hardcoded keybindings
- **Prohibited commands** — never run `npm run dev`, `npm run build`, `npm test` directly
- **Git rules for parallel agents** — never use `git add -A`, never use `git reset --hard`, always stage specific files
- **PR workflow** — analyze PRs without pulling locally, create feature branches, merge into main
- **Changelog format** — breaking changes, added, changed, fixed, removed sections under `[Unreleased]`
- **Style** — no emojis in commits, no fluff, technical prose only
- **Critical tool usage** — never use sed/cat to read files, always use the read tool
- **Adding a new LLM provider** — step-by-step across 7 file groups

This document is not just documentation — it's the instruction set that governs how AI coding agents (Copilot, Claude, pi itself) behave inside the repository. In a Githubified pi-mono, this file would become the system prompt for the agent running on GitHub Actions. The rules already assume multi-agent concurrency and enforce the discipline needed for safe autonomous operation.

### 4. Contributor Gating as Human-in-the-Loop Security

Pi-mono implements a three-workflow contributor management system that is documented in `.WORKFLOW-DESIGN-THEORY.md`:

| Workflow | Trigger | Effect |
|----------|---------|--------|
| **PR Gate** (`pr-gate.yml`) | `pull_request_target: opened` | Checks if PR author is a bot, collaborator, or in `APPROVED_CONTRIBUTORS`; auto-closes unauthorized PRs |
| **Approve Contributor** (`approve-contributor.yml`) | `issue_comment: created` | On `lgtm` from a maintainer, adds the issue author to `APPROVED_CONTRIBUTORS` and commits |
| **CI** (`ci.yml`) | Push to main, PRs targeting main | `npm ci → npm run build → npm run check → npm test` in a single sequential job |

This system is not Githubification — it's developer infrastructure. But it demonstrates sophisticated use of the same GitHub primitives that Githubification requires:

- **`pull_request_target`** instead of `pull_request` prevents fork-based circumvention of the gate
- **`APPROVED_CONTRIBUTORS`** is a version-controlled, auditable allowlist stored in git
- **Automated commit-and-push** in `approve-contributor.yml` shows the pattern of a workflow that writes to the repo — the same pattern a Githubification agent uses to persist state
- **Permission checks** via `repos.getCollaboratorPermissionLevel` enforce authorization at the API level

A Githubification layer for pi-mono would coexist with this system, not replace it. The agent would respond to issue comments; the gating system would control who can open PRs. The two concerns are orthogonal.

### 5. Pre-Built Binaries as Githubification Accelerator

The `build-binaries.yml` workflow produces standalone binaries for five platforms:

```
pi-darwin-arm64.tar.gz
pi-darwin-x64.tar.gz
pi-linux-x64.tar.gz
pi-linux-arm64.tar.gz
pi-windows-x64.zip
```

These are compiled via Bun from the coding-agent package and uploaded as GitHub Release assets. For Githubification, this is a critical escape hatch: instead of running `npm install && npm run build` inside a GitHub Actions workflow (which requires Node.js, npm, and the full dependency tree), a Githubification layer could download the pre-built `pi-linux-x64` binary and execute it directly. This reduces the agent startup time from minutes to seconds and eliminates build-time failures.

The IronClaw lesson identified pre-built binaries as an escape hatch for Rust. Pi-mono proves the same pattern applies to TypeScript monorepos: publish to GitHub Releases, download in the workflow, run.

### 6. The `.pi/` Directory as Agent Self-Configuration

Pi-mono includes a `.pi/` directory at the repository root — the same configuration mechanism used by the pi coding agent when it runs inside any repository. This directory contains:

| Subdirectory | Contents | Purpose |
|---|---|---|
| `extensions/` | `diff.ts`, `files.ts`, `prompt-url-widget.ts`, `redraws.ts`, `tps.ts` | Custom tools that extend the agent's capabilities when working on this repo |
| `prompts/` | `cl.md`, `is.md`, `pr.md` | Reusable prompt templates for changelog, issues, and PR workflows |
| `git/` | `.gitignore` | Git configuration for agent sessions |
| `npm/` | `.gitignore` | npm configuration for agent sessions |

This is the agent configuring itself for its own development. The extensions add capabilities specific to monorepo development (diff visualization, file tracking, token-per-second measurement). The prompts provide templates that the agent uses when writing changelogs or analyzing issues.

In a Githubified pi-mono, this directory would serve double duty: it configures the coding agent both for interactive development (a human running `pi` locally) and for autonomous execution (the agent running on GitHub Actions in response to an issue). The configuration is the same because the agent is the same.

### 7. Supply Chain Hardening in Release Automation

The `build-binaries.yml` workflow pins its GitHub Actions to full commit SHAs:

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
- uses: oven-sh/setup-bun@4bc047ad259df6fc24a6c9b0f9a0cb08cf17fbe5 # v2.0.1
- uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af # v4.1.0
```

This is deliberate supply chain hardening: mutable version tags (`@v4`) can be moved by upstream maintainers (or attackers who compromise the upstream repo), but commit SHAs are immutable. Any Githubification layer for pi-mono should adopt the same practice for its own workflow — particularly because the Githubified agent would have write access to the repository, making supply chain attacks against the workflow especially dangerous.

### 8. Test Isolation from LLM Providers

The `test.sh` script implements a pattern critical for CI: it backs up `~/.pi/agent/auth.json`, unsets all LLM provider API keys (27 environment variables across 15+ providers), and then runs `npm test`. Tests that require LLM access are skipped; tests that don't are exercised.

This same pattern applies to Githubification testing. A Githubified pi-mono agent needs LLM keys to operate (stored in GitHub Secrets), but the CI pipeline that validates the Githubification layer itself should run without them. The test isolation pattern is already proven in this repo.

### 9. Pre-Commit Quality Gates via Husky

The `.husky/pre-commit` hook runs `npm run check` (Biome formatting/linting + TypeScript type checking) on every commit and restages any files that were auto-formatted. This ensures that no commit enters the repository without passing the full quality gate.

For Githubification, this is relevant because the agent would make commits during its operation (persisting sessions, posting responses). If the pre-commit hook runs during agent commits, it must pass — meaning the agent's output must conform to the repo's quality standards. This is either a constraint (the agent must produce valid code) or an obstacle (the hook must be bypassed for state-only commits). Pi-mono's existing Husky setup forces this decision to be made explicitly during Githubification.

### 10. Lockstep Versioning Across Packages

All seven packages share the same version number, incremented together via `scripts/release.mjs`. This means a Githubification layer doesn't need to track per-package versions — it can depend on the monorepo version as a single unit. The `scripts/sync-versions.js` ensures internal cross-references stay consistent.

This also means that when GMI or GitClaw depends on `@mariozechner/pi-coding-agent@^0.52.5`, they implicitly depend on the exact same version of `pi-ai`, `pi-agent-core`, and `pi-tui`. Version coherence is guaranteed by the lockstep release process.

---

## What Pi Mono Teaches About the Foundation Layer

Pi-mono occupies a unique position in the Githubification case studies. It is not a Githubified repo. It is not even a repo that is about to be Githubified. It is **the engine that makes Githubification possible** for other repos.

| Dimension | Other Type 1 Subjects | Pi Mono |
|-----------|----------------------|---------|
| **Role** | Agent to be Githubified | Agent that powers Githubification |
| **Complexity** | Single agent (OpenClaw, IronClaw) or single folder (GMI, GitClaw) | 7-package monorepo with shared build chain |
| **Githubification status** | Wrapped, native, or pre-Githubification | Pre-Githubification, but already consumed by Githubified repos |
| **Channel story** | GitHub Issues is the primary channel | CLI, Slack, web — GitHub Issues would be one more adapter |
| **Binary distribution** | Some (IronClaw via cargo-dist) | Yes — Bun-compiled binaries for 5 platforms |
| **Agent configuration** | `.pi/settings.json` or `AGENTS.md` | Both — plus extensions, prompts, and agent-specific tooling |
| **Contributor model** | Standard open-source | Human-in-the-loop gating with automated approval |
| **Documentation philosophy** | READMEs and inline comments | `.WORKFLOW-DESIGN-THEORY.md` explains every workflow's design rationale |

The comparison reveals a pattern: pi-mono has already solved many of the problems that other Githubification layers encounter. Multi-provider LLM support? Built into `pi-ai`. Session management? Built into `pi-agent-core`. Channel abstraction? Proven by `pi-mom`. Binary distribution? Proven by `build-binaries.yml`. Agent self-configuration? Proven by `.pi/`. The foundation layer provides the primitives; Githubification composes them.

---

## Lessons for Any Type 1 Githubification

1. **Study the foundation, not just the wrapper.** Understanding how the pi coding agent works — its CLI interface, environment-variable configuration, filesystem-based state, session management — is essential for building any native-strategy Githubification layer. The agent's design constraints become the Githubification layer's design constraints.

2. **Monorepos are harder to Githubify than single packages.** A monorepo has build-order dependencies, shared TypeScript path mappings, and a multi-stage build process. Githubifying it requires either building from source (slow, fragile) or using pre-built binaries (fast, requires release infrastructure). Pi-mono has the release infrastructure; not every monorepo will.

3. **Pre-built binaries are the escape hatch for complex builds.** The `build-binaries.yml` workflow compiles the coding agent into standalone executables. A Githubification workflow can download and run these directly, bypassing the monorepo build chain entirely. If your agent has a complex build process, invest in binary distribution first.

4. **Channel adapters prove Githubifiability.** Pi-mom demonstrates that the coding agent can be invoked from a non-terminal context (Slack) with text-in/text-out interaction. This is the same interaction model that GitHub Issues provides. If your agent already supports multiple channels, adding a GitHub Issues channel is incremental, not transformative.

5. **AGENTS.md is the proto-system-prompt.** Pi-mono's `AGENTS.md` already governs how AI agents behave inside the repository — code quality rules, git discipline, prohibited commands, parallel-agent safety. In a Githubified version, this file would become the system prompt for the agent running on GitHub Actions, with minimal modification.

6. **Contributor gating and Githubification are orthogonal.** Pi-mono's PR Gate and Approve Contributor workflows manage _human_ access to the codebase. A Githubification agent would manage _AI_ access to the codebase. The two systems coexist because they operate on different triggers (PR opened vs. issue commented) and serve different purposes (contribution quality vs. conversational interaction).

7. **Pin your action SHAs.** Pi-mono's `build-binaries.yml` uses full commit SHA references for all GitHub Actions. Any Githubification layer that grants write access to the repository must do the same — a compromised action could inject malicious code into the agent's commits.

8. **Test without API keys.** Pi-mono's `test.sh` proves the entire test suite runs without LLM provider credentials by explicitly unsetting 27 environment variables. Githubification layers should adopt the same pattern: test the orchestration logic without requiring (or exposing) API keys.

9. **Self-configuration scales to self-Githubification.** The `.pi/` directory already configures the agent for pi-mono's own development. A Githubification layer would inherit this configuration, giving the agent the same extensions, prompts, and tools whether invoked interactively or autonomously. The agent does not need separate configuration for GitHub — it reads the repo it's in.

10. **The foundation layer's quality gates apply to the agent's output.** Pi-mono's pre-commit hook enforces Biome formatting and TypeScript type checking on every commit. A Githubified agent making commits in this repo must produce output that passes these gates, or the Githubification layer must explicitly manage when hooks run. This is a constraint that simpler repos (where the agent only commits Markdown state) do not face.

---

## Summary

`japer-technology/github-pi-mono` is the foundation layer of Githubification. It is the monorepo that produces the `@mariozechner/pi-coding-agent` package — the single runtime dependency that powers every native-strategy Githubified repository studied in this collection. The coding agent's design — stateless CLI invocation, environment-variable configuration, filesystem-based state, multi-provider LLM support, and proven channel abstraction — makes it inherently Githubifiable, even though the monorepo itself has not been Githubified.

Pi-mono's GitHub infrastructure is among the most sophisticated of any repo in this study: a three-workflow contributor gating system, SHA-pinned release automation, pre-commit quality gates, and a `.WORKFLOW-DESIGN-THEORY.md` document that explains every design decision. These are not Githubification patterns, but they demonstrate mastery of the same GitHub primitives that Githubification depends on.

The core insight is one of self-reference: **the agent that makes Githubification possible for others has not been Githubified itself, because it serves a different role — it is the engine, not the vehicle.** Githubifying pi-mono would mean the engine runs on GitHub to develop itself. This is possible (the channel adapter exists, the binary distribution exists, the self-configuration exists), but it transforms the repo from a foundation layer into a recursive system: an AI agent, running on GitHub, developing the AI agent that runs on GitHub. Whether that recursion is valuable or dangerous is the design question that pi-mono's eventual Githubification must answer.
