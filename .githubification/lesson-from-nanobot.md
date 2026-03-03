# Lesson from Nanobot

### What `japer-technology/github-nanobot` teaches us about Type 1 — AI Agent Repo Githubification

---

## The Subject

[nanobot](https://github.com/HKUDS/nanobot) is an ultra-lightweight personal AI assistant framework built in Python. Inspired by [OpenClaw](https://github.com/openclaw/openclaw), it delivers core agent functionality in approximately **~4,000 lines** of code — 99% smaller than Clawdbot's 430k+ lines. The project provides autonomous task execution through LLM-powered reasoning, a modular tool system (shell execution, filesystem, web search, MCP, subagent spawning), persistent memory, scheduled tasks via cron and heartbeat, a skill system, and — its defining feature — a **multi-channel gateway** that connects the agent to Telegram, Discord, WhatsApp, Slack, Feishu, DingTalk, Email, QQ, Matrix, and Mochat simultaneously. nanobot supports 17+ LLM providers, Docker deployment, and a clean CLI. The project is MIT-licensed, research-oriented, and actively developed.

`japer-technology/github-nanobot` is the Githubified version. The repository contains nanobot's full source — the Python agent core, the Node.js WhatsApp bridge, the Docker configuration, the test suite — exactly as the upstream project ships it. Githubification aims to take this agent, which was designed to be installed via `pip install nanobot-ai` and run as a persistent gateway process, and make it **executable directly on GitHub using GitHub Actions**.

This is a Type 1 — AI Agent Repo Githubification that reveals a critical architectural fault line: **the agent's core reasoning loop is lightweight and stateless enough to run on GitHub Actions, but the multi-channel gateway that defines nanobot's user experience requires persistent, long-running connections that GitHub Actions cannot provide.**

---

## The Core Lesson

> **When an AI agent has a clean separation between its reasoning core and its communication channels, Githubification can wrap the core while replacing the channels — and the result is a genuine execution of the original agent, not a substitution.**

The Agent Zero lesson demonstrated substitution: the agent was too complex to run, so a different agent was deployed alongside it. nanobot teaches a different path. Its architecture cleanly separates the **agent loop** (receive message → reason with LLM → call tools → return response) from the **channels** (Telegram, Discord, WhatsApp, etc.) that deliver messages to and from users. The agent loop does not know or care where the message came from. It processes input and produces output.

This separation means Githubification does not need to substitute. It can **run nanobot's actual agent loop** on GitHub Actions, with GitHub Issues as the communication channel. The agent loop is the same code that processes Telegram messages — it just receives its input from an issue comment instead of a Telegram webhook. The tools are the same. The LLM reasoning is the same. The memory system is the same. The skills are the same.

| GitHub Primitive | Maps To |
|---|---|
| **GitHub Actions** | Compute — the runner that installs nanobot and executes the agent loop |
| **Git** | Storage and memory — sessions, workspace state, and memory files are committed |
| **GitHub Issues** | User interface — each issue is a conversation thread, replacing Telegram/Discord/etc. |
| **GitHub Secrets** | Credential store — LLM API keys that would normally live in `~/.nanobot/config.json` |

The multi-channel gateway — the persistent process that maintains WebSocket connections to Telegram, Discord, WhatsApp, and other platforms — cannot run. But the agent itself can.

---

## Anatomy of the Repo

The repository has two distinct layers:

### Layer 1: nanobot Source (Untouched)

The entire nanobot codebase remains in the repository exactly as the upstream project ships it. The core structure:

```
nanobot/
├── agent/                  # 🧠 Core agent logic (~4,000 lines)
│   ├── loop.py             #    Agent loop — LLM ↔ tool execution cycle
│   ├── context.py          #    Prompt builder — assembles system/user prompts
│   ├── memory.py           #    Persistent memory — long-term recall
│   ├── skills.py           #    Skills loader — modular capability packages
│   ├── subagent.py         #    Background task execution — spawn child agents
│   └── tools/              #    Built-in tools
│       ├── shell.py        #      Shell command execution (with safety guards)
│       ├── filesystem.py   #      File read/write/edit/list
│       ├── web.py          #      Web search and page fetching
│       ├── cron.py         #      Scheduled task management
│       ├── mcp.py          #      Model Context Protocol integration
│       ├── spawn.py        #      Subagent spawning
│       └── message.py      #      Message delivery to channels
├── channels/               # 📱 Chat channel integrations
│   ├── telegram.py         #    Telegram bot (python-telegram-bot)
│   ├── discord.py          #    Discord bot
│   ├── whatsapp.py         #    WhatsApp via Node.js bridge
│   ├── slack.py            #    Slack Socket Mode
│   ├── feishu.py           #    Feishu WebSocket
│   ├── dingtalk.py         #    DingTalk Stream Mode
│   ├── email.py            #    IMAP/SMTP email
│   ├── qq.py               #    QQ via botpy
│   ├── matrix.py           #    Matrix/Element
│   ├── mochat.py           #    Mochat Socket.IO
│   ├── manager.py          #    Channel lifecycle management
│   └── base.py             #    Channel base class
├── bus/                    # 🚌 Message routing between channels and agent
├── cron/                   # ⏰ Scheduled task engine (croniter-based)
├── heartbeat/              # 💓 Proactive wake-up — periodic task execution
├── providers/              # 🤖 LLM provider registry (17+ providers)
├── session/                # 💬 Conversation session management
├── config/                 # ⚙️ Configuration schema (Pydantic)
├── cli/                    # 🖥️ CLI commands (Typer)
├── skills/                 # 🎯 Bundled skills
│   ├── github/             #    GitHub integration
│   ├── weather/            #    Weather queries
│   ├── tmux/               #    Terminal multiplexer
│   ├── cron/               #    Cron task templates
│   ├── memory/             #    Memory management
│   ├── summarize/          #    Content summarization
│   ├── clawhub/            #    ClawHub skill marketplace
│   └── skill-creator/      #    Meta-skill for creating new skills
├── templates/              # 📝 Prompt templates (AGENTS.md, SOUL.md, etc.)
└── utils/                  # 🔧 Shared utilities
bridge/                     # 🌉 WhatsApp Node.js bridge (TypeScript)
├── src/                    #    Bridge source
├── package.json            #    Node.js dependencies (whatsapp-web.js, ws)
└── tsconfig.json           #    TypeScript config
tests/                      # 🧪 Test suite (17+ test files)
pyproject.toml              # 📦 Package definition (~25 Python dependencies)
Dockerfile                  # 🐳 Multi-stage Docker build
docker-compose.yml          # 🐳 Docker orchestration
core_agent_lines.sh         # 📏 Line count verification script
```

The separation between `agent/` and `channels/` is the architectural feature that makes Githubification feasible. The agent does not import from channels. The channels import from bus and agent. The message bus mediates all communication. This means you can run the agent loop without starting any channel — and that is exactly what GitHub Actions can do.

### Layer 2: Infrastructure and Deployment

The repository includes multiple deployment mechanisms — `pip install`, Docker, docker-compose, systemd service — all designed for persistent, long-running execution:

| Deployment Method | Execution Model | GitHub Actions Compatible? |
|---|---|---|
| `pip install nanobot-ai` + `nanobot agent -m "..."` | Single-shot CLI | ✅ Yes — one message, one response |
| `nanobot gateway` | Persistent process, multi-channel | ❌ No — requires long-running daemon |
| Docker / docker-compose | Persistent container | ❌ No — requires long-running container |
| systemd service | Persistent daemon | ❌ No — requires persistent host |

The single-shot CLI mode (`nanobot agent -m "message"`) is the key to Githubification. It processes one message through the full agent loop — LLM reasoning, tool execution, memory — and exits. This mode maps perfectly to the GitHub Actions execution model: one issue comment triggers one workflow run, which processes one message and posts the response.

---

## Key Patterns

### 1. The Agent-Channel Separation Principle

nanobot's most important architectural decision for Githubification is the clean boundary between the agent core and the communication channels. The agent loop (`agent/loop.py`) processes a message through a cycle:

1. Build context (system prompt + memory + skills + conversation history)
2. Send to LLM
3. If LLM requests tool calls, execute them
4. Repeat until LLM produces a final response
5. Save to session, update memory

This loop is completely channel-agnostic. It receives a string and returns a string. Whether the string came from Telegram, Discord, WhatsApp, or a GitHub Issue is irrelevant to the agent. The message bus (`bus/`) handles routing, but the agent itself is stateless with respect to the communication channel.

This is the inverse of Agent Zero's architecture, where the agent was deeply coupled to its WebSocket-based web UI and could not easily be separated from it. nanobot's agent can run anywhere because it was designed to run everywhere — across 10+ different chat platforms. That design for portability is what makes Githubification feasible.

### 2. Lightweight Dependency Footprint

nanobot's Python dependencies are substantial (~25 packages in `pyproject.toml`) but not prohibitive:

| Category | Dependencies | Cold Start Impact |
|---|---|---|
| Core | `litellm`, `pydantic`, `httpx`, `loguru`, `rich` | ~30 seconds |
| Channels | `python-telegram-bot`, `slack-sdk`, `lark-oapi`, `dingtalk-stream`, etc. | Installed but unused on Actions |
| Tools | `croniter`, `readability-lxml`, `mcp` | Minimal |
| CLI | `typer`, `prompt-toolkit` | Minimal |

Compare this to Agent Zero's 54 dependencies, Playwright, Tesseract, and FAISS. nanobot can be installed on a GitHub Actions runner in under a minute via `pip install nanobot-ai`. The channel dependencies (Telegram, Discord, etc.) are installed but never imported when running in single-shot CLI mode — Python's lazy imports mean they add installation time but not runtime cost.

This is a critical lesson: **dependency weight determines whether wrapping or substitution is the right Githubification strategy.** Agent Zero's dependency weight forced substitution. nanobot's dependency weight allows wrapping.

### 3. Single-Shot CLI as the Githubification Entry Point

nanobot provides a CLI command that processes exactly one message:

```bash
nanobot agent -m "What is the weather in Hong Kong?"
```

This command:
1. Loads configuration from `~/.nanobot/config.json`
2. Initializes the agent loop
3. Processes the message through the full LLM reasoning + tool execution cycle
4. Prints the response
5. Exits

This is a ready-made entry point for GitHub Actions. A workflow triggered by an issue comment can:
1. Install nanobot (`pip install nanobot-ai`)
2. Write a `config.json` with API keys from GitHub Secrets
3. Run `nanobot agent -m "$ISSUE_COMMENT_BODY"`
4. Capture the output
5. Post it as an issue comment via `gh api`

No lifecycle scripts needed. No custom orchestrator. The agent's own CLI provides the stateless, single-invocation interface that GitHub Actions requires. This is a pattern not seen in the other lesson repos: **when the agent already has a CLI mode, the Githubification layer can be remarkably thin.**

### 4. Memory Persistence via Git

nanobot stores persistent memory in `~/.nanobot/workspace/` — a directory of Markdown and JSON files that the agent reads and writes across sessions. The `HEARTBEAT.md` file contains periodic tasks. The `AGENTS.md` template defines the agent's personality. Memory consolidation happens via LLM summarization.

On GitHub Actions, this workspace maps to the repository itself. Session state and memory files can be committed to git after each interaction, providing persistence across ephemeral workflow runs. This is the same git-as-memory pattern used by GitClaw, GMI, and Issue Intelligence — but here it applies to an agent that was designed for filesystem-based memory, making the mapping natural rather than forced.

### 5. Multi-Provider Architecture as Githubification Enabler

nanobot supports 17+ LLM providers through a clean provider registry (`providers/registry.py`). Each provider is a `ProviderSpec` with auto-detection, environment variable mapping, and LiteLLM integration. Users configure their preferred provider in `config.json`.

For Githubification, this means the agent is not locked to any particular LLM service. Users store their API key in a GitHub Secret, the workflow writes it to `config.json`, and nanobot's provider system handles the rest. Whether the user wants OpenRouter, Anthropic, OpenAI, DeepSeek, Groq, or a self-hosted vLLM instance — the same Githubification layer works for all of them.

### 6. Skills as Portable, GitHub-Compatible Units

nanobot's skill system uses Markdown files (following the SKILL.md standard) that are loaded into the agent's context at runtime. Skills are directories containing Markdown descriptions and optional shell scripts:

```
nanobot/skills/
├── github/           # GitHub API operations
├── weather/          # Weather queries
├── memory/           # Memory management
├── cron/             # Scheduled tasks
├── summarize/        # Content summarization
├── clawhub/          # ClawHub marketplace
├── skill-creator/    # Create new skills
└── tmux/             # Terminal multiplexer
```

These skills are text files. They don't require compilation, special runtimes, or external services. They are loaded from the filesystem and injected into the LLM prompt. This means they work identically on a local machine and on a GitHub Actions runner — no adaptation needed.

The `github/` skill is particularly relevant: nanobot already has built-in knowledge of GitHub operations. When Githubified, this skill gives the agent native understanding of the platform it's running on.

### 7. Security Model That Transfers to Actions

nanobot's security model includes:

- **Dangerous command blocking** — patterns like `rm -rf /`, fork bombs, and filesystem formatting are blocked in `shell.py`
- **Path traversal protection** — file operations validate paths to prevent escape
- **Workspace sandboxing** — `restrictToWorkspace: true` constrains all file operations
- **Channel allowFrom lists** — restrict which users can interact
- **Command execution timeouts** — 60-second default prevents hanging

On GitHub Actions, these protections layer with GitHub's own security model:

| nanobot Security | GitHub Actions Security |
|---|---|
| Command blocking | Runner isolation (ephemeral VMs) |
| Path traversal protection | Repository-scoped filesystem |
| Workspace sandboxing | Working directory scoping |
| allowFrom lists | Workflow permission checks (collaborator status) |
| Execution timeouts | Job timeout limits (6 hours max) |

The two security models are complementary. nanobot's built-in guards handle LLM-generated command safety. GitHub Actions' infrastructure handles isolation and access control. Neither model needs to be modified for the other to work.

### 8. The Gateway Boundary: What Cannot Be Githubified

The multi-channel gateway (`nanobot gateway`) is nanobot's most visible feature and the one that fundamentally cannot run on GitHub Actions. The gateway:

| Gateway Requirement | GitHub Actions Constraint |
|---|---|
| Maintains persistent WebSocket connections to Telegram, Discord, Slack, Feishu, DingTalk | No persistent connections — each run is ephemeral |
| Keeps WhatsApp session alive via Node.js bridge on localhost:3001 | No persistent processes between workflow runs |
| Polls IMAP for incoming email continuously | No long-running polling loops |
| Runs heartbeat tasks every 30 minutes | Scheduled workflows have minimum 5-minute intervals and no sub-minute precision |
| Manages concurrent sessions across channels | Each workflow run is isolated — no shared state |

This is a hard architectural boundary. The gateway is a long-running daemon. GitHub Actions runs ephemeral jobs. You cannot force-fit a daemon into a batch system.

But this is also where nanobot's architecture shines: the gateway is a **separate concern** from the agent. Losing the gateway means losing multi-platform reach, not losing intelligence. The agent loop, tools, skills, memory, and LLM reasoning all work without the gateway. The Githubified nanobot trades breadth of communication (10+ platforms) for depth of integration (GitHub-native).

---

## What Nanobot Teaches That the Other Lessons Don't

### 1. Wrapping Is Feasible When the Agent Is Lightweight and Modular

Agent Zero required substitution because its architecture was monolithic and heavy. nanobot demonstrates the opposite: a lightweight, modular agent where the core reasoning loop can be extracted from its deployment context and run in isolation. The ~4,000-line core, ~25 Python dependencies, and single-shot CLI mode make genuine wrapping possible. **The threshold for wrapping vs. substitution is not just about complexity — it's about modularity.**

### 2. The CLI Is the Githubification Interface

Most AI agents are designed for interactive, streaming, long-running use. nanobot's `nanobot agent -m "..."` command provides a batch-mode interface that maps directly to the GitHub Actions execution model. This single command replaces the need for custom lifecycle scripts, orchestrators, and agent adapters. **If your agent has a CLI that processes one message and exits, your Githubification layer can be a shell script.**

### 3. Channel Agnosticism Enables Platform Migration

nanobot's support for 10+ communication channels was designed for user convenience, but it has a deeper architectural consequence: the agent was forced to be channel-agnostic. No channel-specific logic leaked into the core. This channel-agnosticism is exactly what Githubification needs — it treats GitHub Issues as "just another channel." **Building for many platforms makes it easier to add one more, even when that platform is GitHub itself.**

### 4. Dependency Weight Is a Githubification Metric

The feasibility of wrapping an agent on GitHub Actions is directly proportional to the speed of installing its dependencies. nanobot's `pip install nanobot-ai` takes under a minute on a standard runner. Agent Zero's `pip install -r requirements.txt` (54 packages including Playwright and Tesseract) takes 15-20 minutes. At some threshold, cold start time makes wrapping impractical and substitution necessary. **Measure your agent's install time on a clean runner before deciding your Githubification strategy.**

### 5. The Ultra-Lightweight Philosophy Transfers

nanobot's guiding principle — deliver core agent functionality in minimal code — is independently valuable for Githubification. Fewer lines of code mean fewer dependencies, faster installation, smaller attack surface, easier auditing, and simpler debugging on ephemeral runners. The "99% smaller than Clawdbot" positioning is not just a marketing claim — it is an architecture decision that makes the agent viable on constrained infrastructure. **Lightweight agents are inherently more Githubifiable.**

### 6. Existing Skills Bootstrap the GitHub-Native Agent

When nanobot runs on GitHub Actions, it brings its full skill system: GitHub operations, web search, file management, memory, cron scheduling, content summarization, and the skill-creator meta-skill. The agent is not starting from zero — it arrives with a portfolio of capabilities that are immediately useful in the repository context. **Skills designed for general use often map surprisingly well to the GitHub-as-infrastructure context.**

---

## Feasibility Assessment

A component-by-component evaluation of nanobot against GitHub Actions' constraints:

| Component | Feasibility | Notes |
|---|---|---|
| Agent loop (loop.py) | ✅ Highly feasible | Stateless per-turn, external LLM API calls, fits ephemeral model perfectly |
| Tool system (shell, filesystem, web) | ✅ Highly feasible | All tools work on standard runners; shell guards prevent dangerous commands |
| Skills system | ✅ Highly feasible | Markdown files loaded from filesystem — no adaptation needed |
| Memory (workspace files) | ✅ Highly feasible | Git commit after each run provides persistent memory across sessions |
| Provider system (17+ LLMs) | ✅ Highly feasible | API key from Secrets, provider auto-detection handles the rest |
| MCP integration | ⚠️ Partially feasible | Stdio MCP servers work; remote HTTP servers depend on network access |
| Subagent spawning | ⚠️ Partially feasible | Works within a single run; no cross-run coordination |
| Cron / Heartbeat | ⚠️ Partially feasible | Scheduled workflows replace `croniter`; minimum 5-minute interval, no sub-minute precision |
| Session management | ⚠️ Partially feasible | Session files committed to git; no concurrent session support |
| Multi-channel gateway | ❌ Not feasible | Requires persistent daemon with WebSocket/long-poll connections |
| WhatsApp bridge | ❌ Not feasible | Requires persistent Node.js process, QR authentication, localhost binding |
| Real-time streaming | ❌ Not feasible | No bidirectional connection to deliver token-by-token responses |

**Overall Verdict:** Substantially feasible. nanobot's core agent — reasoning loop, tools, skills, memory, and provider system — can run on GitHub Actions via the existing `nanobot agent -m` CLI command. The multi-channel gateway cannot, but the agent's value does not depend on the gateway. The Githubified nanobot trades multi-platform reach for GitHub-native integration, and the trade is favorable: the agent's intelligence, tools, and skills are preserved completely.

---

## Lessons for Any Type 1 Githubification

1. **Look for the agent-channel boundary.** The most important architectural feature for Githubification is a clean separation between the agent's reasoning core and its communication layer. If the agent can process a message without knowing where it came from, you can replace the channel with GitHub Issues without touching the core.

2. **Check for CLI batch mode.** If the agent has a command that processes one message and exits, your Githubification layer can be as thin as a workflow file. You don't need custom orchestrators or lifecycle scripts — the agent's own interface provides the stateless invocation that Actions requires.

3. **Measure cold start time.** Install the agent from scratch on a clean Ubuntu runner and time it. Under 2 minutes: wrapping is practical. 2-10 minutes: wrapping is feasible but consider caching. Over 10 minutes: substitution may be the better path.

4. **Evaluate dependency weight honestly.** Count the dependencies. Check for heavy native packages (FAISS, Playwright, Tesseract). Check for Node.js/system dependencies that need separate installation. The lighter the footprint, the faster the cold start, the more practical wrapping becomes.

5. **Map the gateway boundary explicitly.** Identify every component that requires a persistent process, inbound network connections, or long-running state. These are the components that cannot be Githubified. Document them clearly and ensure the remaining components provide sufficient value without them.

6. **Skills and memory transfer naturally.** If the agent has a skill system based on text files and a memory system based on local files, both map directly to the git-as-storage model. Skills become repository files. Memory becomes committed state. No adaptation needed.

7. **Leverage existing security controls.** If the agent already has command blocking, path traversal protection, and timeout limits, these controls complement GitHub Actions' runner isolation. You don't need to reimplement security — you need to verify that the existing controls work on the Actions runner.

8. **The four primitives still apply.** Whether the original agent runs on Telegram, Discord, WhatsApp, or a local terminal, the Githubification mapping is the same: Actions as compute, Git as memory, Issues as UI, Secrets as credentials. Universality of this mapping is confirmed again.

9. **Lightweight beats complex.** nanobot's ~4,000-line core proves that agent capability and code weight are not proportional. A lightweight agent installs faster, starts faster, uses less memory, and is easier to audit on ephemeral infrastructure. When designing agents with Githubification in mind, minimize code weight ruthlessly.

10. **Channel-agnostic design is future-proof.** nanobot supports 10+ channels because its agent doesn't know what channel it's talking to. This architectural decision, made for multi-platform support, is exactly what enables Githubification. Design your agent to be channel-agnostic and you get Githubification readiness as a free benefit.

---

## Summary

`japer-technology/github-nanobot` demonstrates what happens when Githubification encounters a lightweight, well-architected AI agent with a clean boundary between its reasoning core and its communication channels. nanobot's ~4,000-line agent loop, modular tool and skill system, multi-provider LLM support, and single-shot CLI mode make genuine **wrapping** feasible — the actual nanobot agent code runs on GitHub Actions, with Issues replacing Telegram/Discord/WhatsApp as the communication channel.

Where Agent Zero teaches substitution (the agent was too complex to wrap), nanobot teaches **selective wrapping**: the core agent runs natively, the multi-channel gateway is acknowledged as infeasible, and the result is a Githubified agent that preserves the original's intelligence, tools, skills, and memory while trading multi-platform reach for GitHub-native integration.

The critical lesson is architectural: **an agent's Githubifiability is determined not by its capability but by its modularity.** nanobot is a capable AI assistant with 10+ channel integrations, 17+ LLM providers, and a full tool suite. But because these capabilities are cleanly separated — agent core from channels, channels from providers, tools from skills — the agent can be extracted from its deployment context and run on a completely different platform. The lightweight philosophy (99% smaller than Clawdbot) is not just about elegance — it is the engineering discipline that makes Githubification practical.

This is what Type 1 Githubification looks like when the agent is small enough to wrap: **you don't substitute — you run the real thing, and you let GitHub be the channel.**
