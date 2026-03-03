# Lesson from IronClaw

### What `japer-technology/github-ironclaw` teaches us about Type 1 — AI Agent Repo Githubification

---

## The Subject

[IronClaw](https://github.com/nearai/ironclaw) is a secure personal AI assistant written in Rust. It is a ground-up reimplementation of [OpenClaw](https://github.com/openclaw/openclaw) — the same TypeScript-based agent studied in `lesson-from-openclaw.md` — rebuilt in a systems language for native performance, memory safety, and single-binary deployment. IronClaw provides multi-channel messaging (REPL, HTTP webhooks, WASM-based Telegram/Slack/Discord/WhatsApp channels, and a web gateway with SSE/WebSocket streaming), a WASM sandbox for untrusted tool execution with capability-based permissions, hybrid full-text + vector search over a persistent workspace, Docker-based container orchestration for isolated job execution, routines (cron, event, webhook) for background automation, a heartbeat system for proactive monitoring, dynamic tool building at runtime, and defense-in-depth security including prompt injection detection, content sanitization, credential protection, and endpoint allowlisting.

`japer-technology/github-ironclaw` is the repository as it exists today — **not yet Githubified**. There is no `.GITIRONCLAW/` folder, no issue-triggered workflow, no Githubification layer. The repository uses GitHub for standard CI/CD: code style checks (formatting + clippy), unit and integration tests across multiple feature configurations, Docker image builds, Telegram channel tests, automated release management via `release-plz` and `cargo-dist`, and PR labeling. GitHub Actions compiles the Rust binary, runs the test suite, and produces cross-platform release artifacts — but the agent itself does not execute on GitHub in response to issues.

This makes IronClaw a **pre-Githubification Type 1 subject**: a full AI agent that runs locally or on a server, studied at the moment before a Githubification layer would be applied. The question is not "what does the Githubification look like?" but "what does the agent's architecture tell us about how Githubification _should_ be done?"

---

## The Core Lesson

> **A systems-language reimplementation of an already-Githubified agent reveals whether Githubification patterns are language-independent — and IronClaw proves they are, if the architecture carries forward the right escape hatches.**

OpenClaw was Githubified by wrapping it with TypeScript lifecycle scripts that ran on Node.js inside GitHub Actions. The wrapping was straightforward because the agent and the wrapper shared the same runtime. IronClaw breaks that assumption: the agent is a compiled Rust binary that requires PostgreSQL with pgvector, a WASM runtime, and optionally Docker for sandboxed execution.

Yet every structural pattern from the OpenClaw Githubification still applies — sentinel guard, lifecycle pipeline, issue-driven conversation, git-as-memory. What changes is the mechanism, not the architecture. And IronClaw's own design decisions provide the escape hatches that make the mechanism viable:

| Challenge | IronClaw's Escape Hatch |
|---|---|
| PostgreSQL is heavy for ephemeral Actions runners | **libSQL feature** — compile with `--no-default-features --features libsql` for an embedded database with no external dependency |
| Rust must be compiled before it can run | **cargo-dist releases** — pre-built binaries for Linux, macOS, and Windows are published to GitHub Releases; download and run, no compilation needed |
| Tool execution needs isolation | **WASM sandbox** — runs natively on the Actions runner without Docker, with capability-based permissions |
| GitHub Issues is not a supported channel | **Multi-channel architecture** — the same adapter pattern used for Telegram, Slack, Discord, and WhatsApp can accommodate a new GitHub Issues adapter |

The lesson: Githubification patterns are portable across languages and runtimes. What makes an agent Githubifiable is not the language it's written in but the architectural choices it makes — and the best choices are the ones that provide lightweight alternatives to heavyweight infrastructure.

---

## Anatomy of the Repository

IronClaw is a conventional, well-structured Rust project — not a Githubification folder, but a full codebase:

```
github-ironclaw/
├── .claude/commands/                  # Claude Code development commands
├── .github/
│   ├── labeler.yml                    # PR auto-labeling rules
│   ├── scripts/
│   │   ├── create-labels.sh           # Label management
│   │   └── pr-labeler.sh              # PR classification
│   └── workflows/
│       ├── code_style.yml             # Formatting (cargo fmt) + Clippy lints
│       ├── e2e.yml                    # End-to-end tests
│       ├── pr-label-classify.yml      # PR label classification
│       ├── pr-label-scope.yml         # PR scope labeling
│       ├── release-plz.yml            # Automated release PRs and publishing
│       ├── release.yml                # Cross-platform binary builds via cargo-dist
│       └── test.yml                   # Unit/integration tests across feature configs
├── AGENTS.md                          # Agent rules (feature parity update policy)
├── CLAUDE.md                          # Development guide (~34KB of architecture docs)
├── CONTRIBUTING.md                    # Contribution rules (feature parity requirement)
├── FEATURE_PARITY.md                  # OpenClaw ↔ IronClaw tracking matrix
├── Cargo.toml                         # Workspace manifest with feature gates
├── Cargo.lock                         # Locked dependency graph
├── Dockerfile                         # Multi-stage build for cloud deployment
├── Dockerfile.worker                  # Worker container for Docker sandbox
├── docker-compose.yml                 # Local dev (PostgreSQL + pgvector)
├── build.rs                           # Build script
├── src/
│   ├── main.rs                        # CLI entry point (clap)
│   ├── lib.rs                         # Library root — module declarations
│   ├── app.rs                         # Application initialization
│   ├── agent/                         # Agent loop, intent routing
│   ├── channels/                      # Channel adapters (REPL, HTTP, WASM, Web)
│   ├── cli/                           # Command-line interface
│   ├── config/                        # Configuration management
│   ├── context/                       # Context window management
│   ├── db/                            # Database abstraction (PostgreSQL + libSQL)
│   ├── extensions/                    # WASM extension loading
│   ├── history/                       # Conversation history
│   ├── hooks/                         # Hook-based interception
│   ├── llm/                           # LLM provider abstraction (via rig-core)
│   ├── observability/                 # Tracing and logging
│   ├── orchestrator/                  # Docker sandbox orchestration
│   ├── pairing/                       # Device pairing (Telegram DM)
│   ├── registry/                      # Tool and channel registry
│   ├── safety/                        # Prompt injection defense, sanitization
│   ├── sandbox/                       # WASM sandbox (wasmtime)
│   ├── secrets/                       # AES-256-GCM secret management
│   ├── setup/                         # Onboarding wizard
│   ├── skills/                        # Skill system (Markdown-based)
│   ├── tools/                         # Built-in tool implementations
│   ├── worker/                        # Job execution workers
│   └── workspace/                     # Persistent memory (hybrid search)
├── channels-src/                      # WASM channel source (Discord, Slack, Telegram, WhatsApp)
├── tools-src/                         # WASM tool source (GitHub, Gmail, Google suite, Okta, Slack, Telegram, web search)
├── wit/
│   ├── channel.wit                    # WIT interface for channel plugins
│   └── tool.wit                       # WIT interface for tool plugins
├── registry/                          # Pre-built WASM bundles (channels + tools)
├── migrations/                        # Database migrations
├── deploy/                            # Cloud deployment (systemd, Cloud SQL proxy)
├── skills/                            # Skill packages (e.g. web-ui-test)
├── tests/                             # Integration tests (config, heartbeat, WASM, workspace, WebSocket, etc.)
├── scripts/                           # Build and utility scripts
└── docs/                              # Extended documentation (LLM providers, Telegram setup, building channels)
```

The absence of a Githubification folder is itself informative. Everything here is designed for local installation or server deployment. The CI workflows validate the code; they do not execute the agent on behalf of users.

---

## Key Patterns

### 1. Dual-Database Architecture

IronClaw's `Cargo.toml` declares two database backends as Cargo features:

```toml
[features]
default = ["postgres", "libsql", "html-to-markdown"]
postgres = [
    "dep:deadpool-postgres",
    "dep:tokio-postgres",
    ...
    "dep:pgvector",
]
libsql = ["dep:libsql"]
```

The `postgres` feature provides production-grade persistence with vector search via pgvector. The `libsql` feature provides an embedded, serverless database that requires no external process. The CI test matrix already validates both configurations:

```yaml
matrix:
  include:
    - name: all-features
      flags: "--all-features"
    - name: default
      flags: ""
    - name: libsql-only
      flags: "--no-default-features --features libsql"
```

For Githubification, the libSQL path is the critical escape hatch. A GitHub Actions runner can compile or download IronClaw with `--no-default-features --features libsql` and run it without setting up PostgreSQL. The embedded database file can be committed to git between workflow runs, providing persistence across sessions — the same pattern used by MicroClaw with SQLite.

### 2. WASM Sandbox for Tool Security

IronClaw's most distinctive architectural feature is its WASM-based tool sandbox. Untrusted tools run in isolated WebAssembly containers powered by Wasmtime:

```
WASM ──► Allowlist ──► Leak Scan ──► Credential ──► Execute ──► Leak Scan ──► WASM
         Validator     (request)     Injector       Request     (response)
```

Each tool gets:
- **Capability-based permissions** — explicit opt-in for HTTP, secrets, and tool invocation
- **Endpoint allowlisting** — HTTP requests only to approved hosts and paths
- **Credential injection** — secrets injected at the host boundary, never exposed to WASM code
- **Leak detection** — scans requests and responses for secret exfiltration attempts
- **Resource limits** — memory, CPU, and execution time constraints

This security model survives Githubification unchanged. The WASM sandbox runs natively on the Actions runner — no Docker required. For Githubification, this is a significant advantage over OpenClaw's approach: tools are sandboxed at the language runtime level rather than at the container level, eliminating the need for Docker-in-Docker on Actions.

### 3. WIT Interface Contracts

IronClaw defines type-safe plugin interfaces using the [WebAssembly Interface Types (WIT)](https://component-model.bytecodealliance.org/design/wit.html) specification:

- `wit/tool.wit` — defines the contract for tool plugins
- `wit/channel.wit` — defines the contract for channel plugins

These contracts mean that adding a new channel (like GitHub Issues) is a matter of implementing the `channel.wit` interface as a WASM component. The host runtime handles loading, capability grants, and lifecycle management. For Githubification, this is the formal version of MicroClaw's channel adapter pattern — not just a software architecture convention, but a **typed, validated interface contract** that ensures every channel adapter meets the same expectations.

### 4. Multi-Channel Architecture

IronClaw currently supports five channel types:

| Channel | Transport | Implementation |
|---|---|---|
| REPL | stdin/stdout | Built-in (`src/channels/`) |
| HTTP | Webhooks | Built-in (`src/channels/`) |
| Web Gateway | SSE + WebSocket | Built-in (`src/channels/`) |
| WASM Channels | Plugin-hosted | Telegram, Slack, Discord, WhatsApp (`channels-src/`) |
| Docker | Container-mediated | Via orchestrator (`src/orchestrator/`) |

The source code for WASM channels lives in `channels-src/`, each as a separate Cargo project that compiles to a WASM component. Pre-built components are shipped in `registry/channels/`. The channel architecture is cleanly separated from the agent core — the agent loop (`src/agent/`) dispatches to channels through a uniform interface.

A GitHub Issues channel adapter would follow the same pattern: implement `channel.wit`, compile to WASM, and register it. The adapter would receive issue comment events as input messages and post issue comments as output. The agent loop would not need to change.

### 5. Single Compiled Binary

IronClaw compiles to a single binary via `cargo build --release`. Unlike OpenClaw (which requires Node.js, npm, and potentially dozens of npm packages at runtime), IronClaw's release artifact is self-contained. The Dockerfile's runtime stage illustrates this:

```dockerfile
FROM debian:bookworm-slim
RUN apt-get install -y --no-install-recommends ca-certificates libssl3
COPY --from=builder /app/target/release/ironclaw /usr/local/bin/ironclaw
```

One binary, two system libraries. For Githubification, this eliminates the dependency management problem entirely. No `npm install`, no `pip install`, no version conflicts, no package-lock drift. Download the binary, make it executable, run it. The lifecycle pipeline shrinks because the "install dependencies" step becomes a single HTTP download from GitHub Releases.

### 6. Feature-Gated Compilation

Cargo features allow IronClaw to be compiled in multiple configurations:

- `default` — PostgreSQL + libSQL + HTML-to-Markdown (full-featured)
- `postgres` — PostgreSQL only (production deployment)
- `libsql` — libSQL only (embedded, serverless)
- `html-to-markdown` — HTML conversion support

For Githubification, this is configuration at the **compilation** level rather than the runtime level. A Githubification layer would specify its feature set once at build time (or download the appropriate pre-built binary), and the resulting binary would contain exactly the capabilities needed for GitHub Actions execution — no more, no less.

### 7. Pre-Built Binary Distribution

IronClaw uses [cargo-dist](https://opensource.axo.dev/cargo-dist/) for cross-platform release automation. The `release.yml` workflow builds binaries for five targets:

```toml
targets = [
    "aarch64-apple-darwin",
    "aarch64-unknown-linux-gnu",
    "x86_64-apple-darwin",
    "x86_64-unknown-linux-gnu",
    "x86_64-pc-windows-msvc",
]
```

And provides four installer formats: shell script, PowerShell, npm, and MSI.

For Githubification, this means the agent does not need to be compiled on the Actions runner. A lifecycle script can download the appropriate pre-built binary from the latest GitHub Release, cache it, and execute it directly. The `x86_64-unknown-linux-gnu` binary runs on standard Ubuntu Actions runners. This transforms the Githubification build step from "install Rust toolchain, compile for 5+ minutes" to "download binary, chmod +x, run."

### 8. Docker-Ready Deployment

IronClaw ships with a multi-stage Dockerfile, a worker Dockerfile for sandboxed execution, a docker-compose configuration with PostgreSQL + pgvector, and systemd service files for cloud deployment. This provides an alternative Githubification path: instead of running the bare binary on the Actions runner, run the Docker container as a service container alongside the workflow.

This Docker path becomes relevant if the Githubification layer needs the full PostgreSQL backend (for vector search, for example). GitHub Actions supports service containers, so:

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg16
    env:
      POSTGRES_DB: ironclaw
      POSTGRES_USER: ironclaw
      POSTGRES_PASSWORD: ${{ secrets.PG_PASSWORD }}
```

The agent container connects to the PostgreSQL service container, and the workflow orchestrates the interaction. This is heavier than the libSQL path but provides full feature parity with the production deployment.

### 9. Heritage Tracking via FEATURE_PARITY.md

IronClaw maintains an explicit, structured tracking document — `FEATURE_PARITY.md` — that maps every OpenClaw feature to its IronClaw implementation status (✅ Implemented, 🚧 Partial, ❌ Not implemented, 🔮 Planned, 🚫 Out of scope, ➖ N/A). This document is not optional: `AGENTS.md` and `CONTRIBUTING.md` both enforce that any PR changing feature behavior must update the parity matrix.

For Githubification, this pattern is instructive beyond IronClaw. When a Githubified agent is reimplemented in a different language, the feature parity document becomes the **specification** for the Githubification layer. It tells the Githubification designer exactly which capabilities are available, which are missing, and which are intentionally excluded. Instead of reverse-engineering the agent's capabilities, the designer reads a structured table.

### 10. CI-Proven GitHub Actions Compilation

IronClaw's `test.yml` workflow already compiles the full Rust project on `ubuntu-latest` runners across three feature configurations and runs the complete test suite. The `code_style.yml` workflow runs formatting and Clippy lint checks. The `release.yml` workflow performs full release builds for five target platforms. This proves that the Rust toolchain, the WASM compilation (for tools and channels), and the complete build pipeline work reliably on GitHub Actions.

For Githubification, this is prior art. The compilation infrastructure is already validated. A Githubification layer does not need to solve "can this agent be built on Actions?" — it only needs to solve "how do we trigger the agent from issues and persist state to git?"

---

## What IronClaw Teaches That OpenClaw Doesn't

The `lesson-from-openclaw.md` document shows how a TypeScript-based AI agent was Githubified by adding a `.GITOPENCLAW/` folder with lifecycle scripts. The agent and the wrapper shared the same language and runtime. IronClaw breaks that symmetry:

1. **Language boundaries reshape wrapping.** OpenClaw's lifecycle scripts were TypeScript calling TypeScript. IronClaw's lifecycle scripts would be shell or TypeScript calling a Rust binary. The wrapper can no longer import the agent's modules or call its functions directly — it must interact through the binary's CLI interface or a channel adapter. This is a cleaner separation: the Githubification layer becomes truly external to the agent.

2. **Compilation introduces a build-or-download decision.** OpenClaw could be installed with `npm install` in seconds. IronClaw requires either a multi-minute Rust compilation or a binary download from GitHub Releases. The pre-built binary path is faster, simpler, and more reliable — but it introduces a version coupling between the Githubification layer and the release artifacts. The lesson: **when Githubifying a compiled agent, always prefer pre-built binaries over source compilation in the workflow.**

3. **Database flexibility is a design-time decision.** OpenClaw's state was managed through files and memory logs — lightweight, git-friendly, and naturally suited to ephemeral runners. IronClaw defaults to PostgreSQL, which requires a persistent server. The libSQL feature was not designed for Githubification — it was designed for embedded deployments — but it serves the Githubification use case perfectly. The lesson: **agents that offer embedded database alternatives are more Githubifiable than those that require external database servers.**

4. **WASM sandbox replaces Docker sandbox.** OpenClaw's Githubification layer needed to consider how tool execution (which sometimes used Docker) would work on Actions. IronClaw solves this at the language level: tools run in WASM containers managed by Wasmtime, with no Docker dependency. This is a Githubification advantage that the Rust rewrite gains for free.

5. **Feature gates enable Githubification-specific builds.** OpenClaw was a single configuration — you got everything or nothing. IronClaw's Cargo features allow compiling a Githubification-optimized binary that includes only what's needed (libSQL, the core agent loop, specific channels) and excludes what's unnecessary (PostgreSQL, Docker orchestrator, web gateway). The resulting binary is smaller, faster to download, and has fewer failure modes.

---

## What IronClaw Teaches That MicroClaw Doesn't

The `lesson-from-microclaw.md` document analyzed another Rust AI agent with multi-channel support. Both IronClaw and MicroClaw are compiled Rust binaries with channel adapter architectures. But IronClaw teaches additional lessons:

1. **A formal plugin interface (WIT) is stronger than a code-level adapter pattern.** MicroClaw's channels are implemented as Rust modules within the same binary. IronClaw's WASM channels are separate WASM components that implement a typed WIT interface. For Githubification, the WIT approach means a GitHub Issues channel could be developed, tested, and deployed independently of the main binary — as a WASM component loaded at runtime.

2. **Pre-built binary distribution solves the compilation problem.** MicroClaw's Githubification plan involved compiling the Rust binary on the Actions runner. IronClaw's cargo-dist infrastructure provides ready-to-download binaries for every release. This is a practical difference: a 30-second binary download versus a 5+ minute compilation on every workflow run.

3. **Heritage tracking provides a specification.** MicroClaw's Githubification design was based on reverse-engineering the agent's capabilities from source code and documentation. IronClaw's `FEATURE_PARITY.md` provides a structured, up-to-date capability matrix that a Githubification designer can read directly.

4. **Dual-database support is built in.** MicroClaw uses SQLite natively, which is already embedded. IronClaw offers both PostgreSQL and libSQL, requiring a conscious choice. The existence of the choice — and the CI validation of both paths — means the Githubification layer can confidently select the lightweight option knowing it is tested.

---

## The Githubification Strategy for IronClaw

Based on IronClaw's architecture, the optimal Githubification strategy combines elements of **Strategy 2 (Wrapping)** and **Strategy 4 (Channel Addition)** from the consolidation document:

### Phase 1 — Binary Wrapping (Immediate Viability)

Add a `.GITIRONCLAW/` folder with lifecycle scripts that:
1. Download the pre-built `x86_64-unknown-linux-gnu` binary from GitHub Releases
2. Configure IronClaw for libSQL (embedded database, no PostgreSQL)
3. Parse issue comments into agent input
4. Invoke the binary via its CLI interface
5. Post agent output as issue comments
6. Commit the libSQL database and workspace files to git

This follows the wrapping pattern from OpenClaw but with a binary interface instead of a module interface.

### Phase 2 — Channel Adapter (Native Integration)

Implement a GitHub Issues channel as a WASM component that implements `channel.wit`:
1. Build a `channels-src/github-issues/` crate that compiles to WASM
2. The adapter translates between GitHub webhook events and the channel interface
3. Register the component in the tool registry
4. IronClaw natively receives and responds to GitHub Issues without lifecycle script mediation

This follows the channel addition pattern from MicroClaw but with a formal WASM plugin rather than a source-code module.

| Phase | Strategy | Lifecycle Scripts | Agent Modification | GitHub Primitive Mapping |
|---|---|---|---|---|
| 1 | Wrapping | Shell/TypeScript scripts invoke the binary | None — binary used as-is | Scripts translate between GitHub and CLI |
| 2 | Channel Addition | Minimal — only guard and orchestration | New WASM channel component | Agent natively speaks GitHub Issues |

---

## Lessons for Any Type 1 Githubification

1. **Provide an embedded database option.** If your agent requires a database, offer a lightweight, serverless alternative alongside the production backend. PostgreSQL is excellent for servers; it is hostile to ephemeral CI runners. libSQL, SQLite, or DuckDB embed cleanly and can be committed to git.

2. **Publish pre-built binaries.** Compiled agents should never require source compilation on the Actions runner. Use cargo-dist, GoReleaser, or equivalent tooling to publish platform-specific binaries to GitHub Releases. The Githubification lifecycle downloads the binary instead of building it.

3. **Use WASM for tool sandboxing when possible.** Docker-based sandboxes are heavy and complex on GitHub Actions. WASM sandboxes run natively on the runner with no privilege escalation, no Docker socket, and no container overhead. If your agent can isolate tools in WASM, do it.

4. **Define typed plugin interfaces.** When the agent supports channel adapters or tool plugins, define those interfaces formally (WIT, protobuf, JSON Schema, or equivalent). A typed interface makes it possible to add a GitHub Issues channel without modifying the agent's core — and to validate the adapter at build time rather than at runtime.

5. **Feature-gate your compilation.** Cargo features, Go build tags, C preprocessor flags — use whatever your language provides to allow Githubification-optimized builds. The Githubification binary should include only what it needs.

6. **Track feature parity explicitly.** If your agent has a heritage (reimplementation, fork, port), maintain a structured parity document. This document becomes the specification for what the Githubification layer can and cannot expose.

7. **Let CI prove the build.** If your agent's existing CI already compiles and tests on GitHub Actions, the Githubification layer inherits that confidence. Don't re-solve build infrastructure — reference the existing workflows and reuse their patterns.

8. **Prefer binary invocation over module import.** When the Githubification wrapper and the agent are in different languages, the CLI is the cleanest interface. Invoke the binary with arguments, read stdout/stderr, check the exit code. This is simpler, more debuggable, and more portable than FFI, IPC, or HTTP.

9. **Design the Githubification in phases.** Start with wrapping (quick, no agent modification). Graduate to native integration (channel adapter) when the wrapping proves the value. The wrapping phase validates the GitHub primitive mapping; the channel adapter phase eliminates the translation overhead.

10. **The agent's architecture is the Githubification specification.** IronClaw's dual-database support, WASM sandbox, multi-channel architecture, and pre-built binary distribution were not designed for Githubification — they were designed for deployment flexibility. But deployment flexibility IS Githubification readiness. An agent that can run in multiple environments can run on GitHub.

---

## Summary

`japer-technology/github-ironclaw` is a Rust reimplementation of OpenClaw — a full AI assistant with WASM-sandboxed tools, multi-channel messaging, hybrid-search persistent memory, Docker orchestration, and defense-in-depth security. It has not yet been Githubified, but its architecture reveals exactly how Githubification should proceed.

The key escape hatches are: **libSQL** for embedded database operation without PostgreSQL, **cargo-dist** for pre-built binary distribution without source compilation, **WASM sandbox** for tool isolation without Docker, and **WIT-defined channel adapters** for adding GitHub Issues as a native channel. These are not Githubification features — they are deployment flexibility features that happen to make Githubification viable.

Where OpenClaw taught that a TypeScript agent can be wrapped with TypeScript lifecycle scripts, IronClaw teaches that the wrapping pattern survives a language boundary. Where MicroClaw taught that a Rust agent's channel architecture naturally accommodates GitHub Issues, IronClaw teaches that a formal plugin interface (WIT) makes that accommodation type-safe and independently deployable.

This is what a **pre-Githubification Type 1 agent** looks like when it is well-architected: the agent does not know about GitHub, but its design decisions — embedded database support, plugin interfaces, single-binary distribution, feature-gated compilation — create all the escape hatches that a Githubification layer needs. The lesson is not about IronClaw specifically. It is about what makes any agent ready for Githubification before the Githubification folder is ever created: **deployment flexibility is Githubification readiness.**
