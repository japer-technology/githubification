# NanoClaw v2 Specification

### Githubification of `japer-technology/github-nanoclaw` — Type 1 AI Agent Repo

---

## Overview

This specification defines the Githubification of [NanoClaw](https://github.com/japer-technology/github-nanoclaw) — a personal Claude assistant that runs as a single Node.js process on macOS/Linux with multi-channel messaging, container-isolated agent execution, per-group memory, and scheduled tasks backed by SQLite.

NanoClaw v2 is the GitHub-native execution model. Instead of running locally via `launchd`/`systemd`, the agent runs on GitHub Actions, uses Git for state, GitHub Issues for user interaction, and GitHub Secrets for credentials. The codebase remains small enough to fit in an AI context window (~35k tokens), and the Githubification layer preserves that property.

**Version:** 2.0  
**Type:** Type 1 — AI Agent Repo  
**Source:** `japer-technology/github-nanoclaw` v1.2.1  
**Strategy:** Channel Addition — GitHub Issues becomes a new channel in NanoClaw's existing channel-agnostic architecture

---

## Table of Contents

1. [Architecture Mapping](#architecture-mapping)
2. [Workflow Design](#workflow-design)
3. [State Management](#state-management)
4. [Channel Adaptation](#channel-adaptation)
5. [Agent Execution Model](#agent-execution-model)
6. [Memory System](#memory-system)
7. [Session Continuity](#session-continuity)
8. [Scheduled Tasks](#scheduled-tasks)
9. [Security Model](#security-model)
10. [Skills Integration](#skills-integration)
11. [Folder Structure](#folder-structure)
12. [Configuration](#configuration)
13. [Implementation Phases](#implementation-phases)

---

## Architecture Mapping

NanoClaw's local architecture maps directly to GitHub primitives. The mapping is clean because NanoClaw was designed for simplicity — one process, six runtime dependencies, and container-based isolation.

### Primitive Mapping

| GitHub Primitive | NanoClaw Local Equivalent | NanoClaw v2 (Githubified) |
|---|---|---|
| **GitHub Actions** | Node.js process + launchd/systemd | Compute — ephemeral workflow runs replace the persistent local process |
| **Git** | SQLite (`store/messages.db`) | Storage — messages, sessions, router state, registered groups committed as JSON/Markdown |
| **GitHub Issues** | WhatsApp / Telegram / Discord / Slack / Gmail channels | User interface — each issue is a conversation thread; comments are messages |
| **GitHub Secrets** | `.env` file with `CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_API_KEY` | Credential store — secrets injected via `${{ secrets.* }}` |

### Pipeline Translation

**Local NanoClaw:**
```
Channel message → SQLite → Polling loop → Container (Claude Agent SDK) → Channel response
```

**NanoClaw v2 (Githubified):**
```
Issue comment → Workflow trigger → Actions runner (Claude Agent SDK) → Issue reply + git commit
```

The polling loop becomes a webhook-triggered workflow. The SQLite queue becomes the GitHub Actions job queue with concurrency control. The container becomes the Actions runner. Channel-specific response routing becomes an issue comment.

### Component Mapping

| NanoClaw Component | File | v2 Equivalent |
|---|---|---|
| Orchestrator | `src/index.ts` | GitHub Actions workflow YAML + lifecycle scripts |
| Channel registry | `src/channels/registry.ts` | GitHub Issues channel adapter (self-registers alongside existing channels) |
| Message router | `src/router.ts` | Lifecycle script that reads issue context and formats prompts |
| Container runner | `src/container-runner.ts` | Direct Claude Agent SDK invocation on the Actions runner (no nested container) |
| Group queue | `src/group-queue.ts` | GitHub Actions `concurrency` groups (one active run per issue) |
| SQLite database | `src/db.ts` | Git-committed JSON state files in `.githubification/state/` |
| Task scheduler | `src/task-scheduler.ts` | `schedule` trigger in GitHub Actions workflow |
| IPC watcher | `src/ipc.ts` | Not needed — no container boundary; agent writes directly to state files |
| Config | `src/config.ts` | Environment variables from GitHub Secrets + workflow inputs |

---

## Workflow Design

### Trigger Events

The workflow responds to two GitHub events:

```yaml
on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]
```

**Issue opened:** Creates a new conversation. The issue title and body become the first user message.

**Issue comment created:** Continues an existing conversation. The comment body is the new user message; all prior comments in the issue provide conversation context.

### Concurrency Control

NanoClaw locally uses a `GroupQueue` with `MAX_CONCURRENT_CONTAINERS` (default: 5) and per-group serialization. In v2, GitHub Actions concurrency groups provide the same guarantees:

```yaml
concurrency:
  group: nanoclaw-${{ github.repository }}-issue-${{ github.event.issue.number }}
  cancel-in-progress: false
```

Each issue gets its own concurrency group. `cancel-in-progress: false` ensures in-flight agent runs complete rather than being cancelled by new comments (matching NanoClaw's queuing behavior where pending messages are processed after the active run finishes).

### Workflow Steps

```yaml
name: nanoclaw-agent

on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]

permissions:
  contents: write
  issues: write

jobs:
  run-agent:
    runs-on: ubuntu-latest
    concurrency:
      group: nanoclaw-${{ github.repository }}-issue-${{ github.event.issue.number }}
      cancel-in-progress: false
    if: >-
      (github.event_name == 'issues')
      || (github.event_name == 'issue_comment'
          && !endsWith(github.event.comment.user.login, '[bot]'))
    steps:
      - name: Authorize
        id: authorize
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PERM=$(gh api "repos/${{ github.repository }}/collaborators/${{ github.actor }}/permission" \
            --jq '.permission' 2>/dev/null || echo "none")
          if [[ "$PERM" != "admin" && "$PERM" != "maintain" && "$PERM" != "write" ]]; then
            echo "::error::Unauthorized: ${{ github.actor }} has '$PERM' permission"
            exit 1
          fi

      - name: Reject unauthorized
        if: ${{ failure() && steps.authorize.outcome == 'failure' }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          if [[ "${{ github.event_name }}" == "issue_comment" ]]; then
            gh api "repos/${{ github.repository }}/issues/comments/${{ github.event.comment.id }}/reactions" \
              -f content=-1
          else
            gh api "repos/${{ github.repository }}/issues/${{ github.event.issue.number }}/reactions" \
              -f content=-1
          fi

      - name: Checkout
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.repository.default_branch }}
          fetch-depth: 0

      - name: Setup runtime
        uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - name: Indicate processing
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: bun .githubification/lifecycle/indicator.ts

      - name: Install dependencies
        run: cd .githubification && bun install --frozen-lockfile

      - name: Run agent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: bun .githubification/lifecycle/agent.ts
```

### Step Pipeline

The workflow follows a five-step pipeline that mirrors the existing GMI pattern:

| Step | Purpose | NanoClaw Local Equivalent |
|---|---|---|
| **Authorize** | Check collaborator permissions | Not applicable (local = trusted) |
| **Indicate** | Add 🚀 reaction for immediate feedback | Channel typing indicator (`setTyping`) |
| **Install** | Install dependencies | `npm install` during setup |
| **Run** | Execute the agent, post reply, commit state | Full message loop cycle |
| **Outcome** | Add 👍/👎 reaction | Agent response sent via channel |

---

## State Management

### From SQLite to Git

NanoClaw locally stores all state in SQLite (`store/messages.db`). In v2, state is committed to Git as structured files. This provides:

- **Persistence** — state survives across ephemeral workflow runs
- **Auditability** — every state change is a git commit with a timestamp
- **Branching** — multiple conversations can run concurrently without conflicts (each issue has its own state file)

### State File Layout

```
.githubification/state/
├── issues/
│   ├── 1.json                    # Issue #1 → session mapping
│   ├── 2.json                    # Issue #2 → session mapping
│   └── ...
├── sessions/
│   ├── <timestamp>.jsonl         # Agent session transcripts (Claude JSONL format)
│   └── ...
├── groups/
│   ├── main/
│   │   └── CLAUDE.md             # Main group memory (admin)
│   └── global/
│       └── CLAUDE.md             # Shared global memory
└── router-state.json             # Last processed timestamps, agent cursors
```

### Issue-to-Session Mapping

Each issue gets a JSON state file:

```json
{
  "issueNumber": 42,
  "sessionFile": "sessions/2026-03-03T235043Z.jsonl",
  "createdAt": "2026-03-03T23:50:43.000Z",
  "lastActivity": "2026-03-04T01:15:00.000Z",
  "groupFolder": "github_issue-42"
}
```

### SQLite Table Translation

| SQLite Table | v2 Equivalent | Format |
|---|---|---|
| `messages` | GitHub Issue comments (API) | Not stored — issue comments are the source of truth |
| `chats` | Issue metadata | Derived from GitHub API at runtime |
| `sessions` | `.githubification/state/issues/<number>.json` | JSON mapping file |
| `registered_groups` | Issues with specific labels (or all issues) | Label-based filtering |
| `scheduled_tasks` | `.githubification/state/scheduled-tasks.json` | JSON array |
| `task_run_logs` | Git commit history + `.githubification/state/task-logs/` | JSON log files |
| `router_state` | `.githubification/state/router-state.json` | JSON key-value |

### Push Conflict Resolution

Multiple concurrent agent runs may attempt to push state changes simultaneously. The lifecycle script implements a retry loop:

1. `git pull --rebase -X theirs` to incorporate remote changes
2. Retry `git push` up to 10 times with exponential backoff
3. State files are per-issue, so conflicts are rare — only `router-state.json` and `scheduled-tasks.json` are shared

---

## Channel Adaptation

### GitHub Issues as a NanoClaw Channel

NanoClaw's channel architecture is designed for exactly this kind of extension. Channels self-register at startup via the factory registry pattern. Adding GitHub Issues as a channel means implementing the `Channel` interface:

```typescript
interface Channel {
  name: string;                                           // "github-issues"
  connect(): Promise<void>;                               // No-op (stateless)
  sendMessage(jid: string, text: string): Promise<void>;  // Post issue comment
  isConnected(): boolean;                                 // Always true in Actions
  ownsJid(jid: string): boolean;                          // Matches "gh:<issue_number>"
  disconnect(): Promise<void>;                            // No-op
  setTyping?(jid: string, isTyping: boolean): Promise<void>; // Add/remove 🚀 reaction
}
```

### JID Format

NanoClaw uses JIDs (Jabber IDs) to identify conversations. The GitHub Issues channel uses the format:

```
gh:<issue_number>
```

Examples: `gh:1`, `gh:42`, `gh:100`

### Message Format

Inbound messages from GitHub Issues are formatted using NanoClaw's existing `formatMessages` function:

```xml
<messages>
<message sender="username" time="2026-03-03T23:50:43Z">Issue body or comment text</message>
</messages>
```

### Trigger Pattern

In the local NanoClaw, messages require a trigger word (default: `@Andy`). In v2, every issue comment is treated as a triggered message — the act of commenting on an issue IS the trigger. This maps to NanoClaw's `requiresTrigger: false` setting that main groups already use.

### Channel Registration

The GitHub Issues channel self-registers alongside existing channels:

```typescript
// src/channels/github-issues.ts
import { registerChannel } from './registry.js';

registerChannel('github-issues', (opts) => {
  // In GitHub Actions context, always available
  if (!process.env.GITHUB_ACTIONS) return null;
  return new GitHubIssuesChannel(opts);
});
```

This means the same NanoClaw codebase can run locally with WhatsApp/Telegram AND on GitHub with Issues — both channels active simultaneously if configured.

---

## Agent Execution Model

### Local vs. GitHub Actions Execution

| Aspect | Local NanoClaw | NanoClaw v2 |
|---|---|---|
| **Process model** | Persistent (launchd/systemd) | Ephemeral (workflow run) |
| **Agent runtime** | Claude Agent SDK in Docker/Apple Container | Claude Agent SDK directly on Actions runner |
| **Isolation** | Container filesystem isolation | Actions runner VM isolation |
| **Concurrency** | `GroupQueue` with `MAX_CONCURRENT_CONTAINERS=5` | Actions `concurrency` groups (1 per issue) |
| **Session resume** | SQLite session table + `data/sessions/` JSONL | Git-committed session files + `--session` flag |
| **Timeout** | `CONTAINER_TIMEOUT` (30 min default) | GitHub Actions job timeout (6 hours max) |

### No Nested Containers

Locally, NanoClaw spawns Docker/Apple Container instances for each agent invocation. On GitHub Actions, the runner IS the isolated environment — it's an ephemeral Ubuntu VM that is destroyed after the workflow completes. There is no need for nested container isolation.

The `container-runner.ts` logic (building volume mounts, spawning containers, managing container lifecycle) is replaced by direct invocation of the Claude Agent SDK via the `pi` coding agent binary or `claude` CLI on the Actions runner.

### Agent Invocation

The lifecycle script invokes the agent with:

```bash
pi --session <session-file> \
   --session-dir .githubification/state/sessions \
   --prompt "<formatted-messages>" \
   --allowedTools "Bash(safe),Read,Write,Edit,Glob,Grep,WebSearch,WebFetch" \
   2>&1 | tee /tmp/agent-raw.jsonl
```

Or equivalently with Claude Code CLI:

```bash
claude --session-id <session-id> \
       --prompt "<formatted-messages>" \
       --output-format json
```

### Tool Availability

The agent on GitHub Actions has access to:

| Tool | Local Container | GitHub Actions Runner |
|---|---|---|
| Bash | ✅ (sandboxed in container) | ✅ (sandboxed in ephemeral VM) |
| Read/Write/Edit/Glob/Grep | ✅ (mounted directories) | ✅ (checked-out repo) |
| WebSearch/WebFetch | ✅ | ✅ |
| Browser automation | ✅ (Chromium in container) | ⚠️ (requires headless Chrome setup) |
| MCP servers | ✅ (nanoclaw MCP for tasks) | ⚠️ (replaced by direct file operations) |

---

## Memory System

### Hierarchical Memory Preserved

NanoClaw's three-level memory hierarchy maps directly to Git:

| Level | Local Path | v2 Path | Access |
|---|---|---|---|
| **Global** | `groups/CLAUDE.md` | `.githubification/state/groups/global/CLAUDE.md` | Read by all issues, written by admin |
| **Per-issue** | `groups/{name}/CLAUDE.md` | `.githubification/state/groups/github_issue-{N}/CLAUDE.md` | Read/written by issue N's agent |
| **Files** | `groups/{name}/*.md` | `.githubification/state/groups/github_issue-{N}/*.md` | Created by the agent |

### Memory Loading

The agent runs with its working directory set to the per-issue group folder. Claude Agent SDK's `settingSources: ['project']` loads:

- `../global/CLAUDE.md` (global memory — read by all)
- `./CLAUDE.md` (issue-specific memory — isolated)

### Memory Persistence

After each agent run, the lifecycle script commits memory changes:

```bash
git add .githubification/state/
git commit -m "nanoclaw: update state for issue #${ISSUE_NUMBER}"
git push
```

---

## Session Continuity

### How Sessions Work

Each issue maintains a conversation session so the agent remembers prior interactions:

1. **First comment** → Create new session, store mapping in `.githubification/state/issues/<N>.json`
2. **Subsequent comments** → Load existing session file, pass to agent via `--session` flag
3. **Agent runs** → Session transcript appended to JSONL file in `state/sessions/`
4. **Post-run** → Updated session file committed to Git

### Session File Format

Sessions are stored as JSONL (JSON Lines) files — one JSON object per line, recording the full conversation transcript. This is the native format used by both the `pi` coding agent and Claude Code CLI.

### Context Window Management

When sessions grow large, the Claude Agent SDK automatically compacts the context — summarizing earlier exchanges while preserving critical information. This behavior is identical between local and Githubified execution.

---

## Scheduled Tasks

### Translation to GitHub Actions Schedules

NanoClaw's task scheduler (`src/task-scheduler.ts`) checks for due tasks every 60 seconds. In v2, scheduled tasks use GitHub Actions' `schedule` trigger:

```yaml
on:
  schedule:
    - cron: '*/5 * * * *'  # Check for due tasks every 5 minutes
```

### Task State

Scheduled tasks are stored in `.githubification/state/scheduled-tasks.json`:

```json
[
  {
    "id": "task-1709510443000-abc123",
    "groupFolder": "github_issue-42",
    "chatJid": "gh:42",
    "prompt": "Send a reminder to review weekly metrics",
    "schedule_type": "cron",
    "schedule_value": "0 9 * * 1",
    "status": "active",
    "next_run": "2026-03-10T09:00:00.000Z",
    "created_at": "2026-03-03T23:50:43.000Z"
  }
]
```

### Task Execution

When the scheduled workflow runs:

1. Read `scheduled-tasks.json` from the repo
2. Find tasks where `next_run <= now` and `status === 'active'`
3. For each due task, run the agent with the task's prompt in the task's group context
4. Post the result as a comment on the task's associated issue
5. Update `next_run` based on schedule type (cron → next occurrence, interval → now + ms, once → null)
6. Commit the updated state

### Schedule Types

| Type | Local Behavior | v2 Behavior |
|---|---|---|
| `cron` | Evaluated every 60s | Evaluated every 5 min via Actions schedule |
| `interval` | Precise millisecond intervals | Quantized to 5-min granularity |
| `once` | ISO timestamp, checked every 60s | Checked every 5 min |

The minimum effective granularity changes from 60 seconds to 5 minutes due to GitHub Actions' minimum cron interval.

---

## Security Model

### Trust Model Translation

| Entity | Local Trust | v2 Trust | Mechanism |
|---|---|---|---|
| Main group | Trusted (self-chat) | Repository admin/maintainer | `gh api` permission check |
| Non-main groups | Untrusted | Write collaborators | Permission check in Authorize step |
| Container agents | Sandboxed | Sandboxed | Ephemeral Actions runner VM |
| User input | Prompt injection risk | Prompt injection risk | Claude safety training + isolation |

### Security Boundaries

**Local NanoClaw** relies on container isolation:
- Process isolation (container boundary)
- Filesystem isolation (explicit mount list)
- Credential filtering (only `CLAUDE_CODE_OAUTH_TOKEN` and `ANTHROPIC_API_KEY`)
- Mount security (external allowlist blocks `.ssh`, `.gnupg`, `.aws`, etc.)

**NanoClaw v2** relies on GitHub Actions isolation:
- VM isolation (ephemeral runner destroyed after each job)
- Filesystem isolation (only the checked-out repository is accessible)
- Credential injection (`${{ secrets.* }}` — available only during the workflow run)
- Permission model (collaborator permission check before agent execution)

### Permission Matrix

| Action | Admin/Maintainer | Write Collaborator | Read/None |
|---|---|---|---|
| Create issue (trigger agent) | ✅ | ✅ | ❌ (rejected with 👎) |
| Comment on issue (continue conversation) | ✅ | ✅ | ❌ (rejected with 👎) |
| Schedule tasks | ✅ | ✅ | ❌ |
| Modify global memory | ✅ | ❌ | ❌ |
| Access repository secrets | ✅ (via agent) | ✅ (via agent) | ❌ |

### Credential Handling

| Credential | Storage | Exposure |
|---|---|---|
| `ANTHROPIC_API_KEY` | GitHub Secret | Injected as env var during workflow |
| `GITHUB_TOKEN` | Auto-generated | Scoped to `contents: write` + `issues: write` |
| Claude OAuth token | GitHub Secret (optional) | Injected as env var during workflow |

---

## Skills Integration

### Githubification as a Skill

NanoClaw's contribution model is "skills over features." Githubification itself should be a skill:

```
.claude/skills/githubify/SKILL.md
```

Running `/githubify` would teach Claude Code how to:

1. Create the `.githubification/` folder structure
2. Add the GitHub Actions workflow file
3. Create the lifecycle scripts (indicator.ts, agent.ts)
4. Initialize state directories
5. Set up `package.json` with runtime dependencies
6. Configure the GitHub Issues channel adapter

### Skill Manifest

The skills engine (`skills-engine/`) supports this via:

- **State tracking** — `.nanoclaw/state.yaml` records the Githubification skill as applied
- **Three-way merge** — if the user has other skills applied, Githubification merges cleanly
- **Uninstall** — running the skill engine's `uninstall` replays all remaining skills from a clean base, removing the Githubification layer
- **Rebase** — flattens accumulated skill layers into a clean starting point

### Composability

The Githubification skill can coexist with channel skills (`/add-whatsapp`, `/add-telegram`). The same NanoClaw installation can run:

- Locally with WhatsApp + Telegram (via launchd/systemd)
- On GitHub with Issues (via Actions workflow)
- Both simultaneously (different channel adapters, shared state via git sync)

---

## Folder Structure

```
.githubification/
├── specification/
│   └── nanoclaw-v2.md             # This specification
├── lifecycle/
│   ├── indicator.ts               # Pre-install: add 🚀 reaction
│   └── agent.ts                   # Main: run agent, post reply, commit state
├── state/
│   ├── issues/                    # Per-issue session mappings
│   │   └── <number>.json
│   ├── sessions/                  # Agent session transcripts (JSONL)
│   │   └── <timestamp>.jsonl
│   ├── groups/
│   │   ├── global/
│   │   │   └── CLAUDE.md          # Shared global memory
│   │   └── github_issue-<N>/
│   │       └── CLAUDE.md          # Per-issue agent memory
│   ├── scheduled-tasks.json       # Scheduled task definitions
│   ├── task-logs/                 # Task execution history
│   └── router-state.json          # Processing cursors and timestamps
├── package.json                   # Runtime dependencies (pi-coding-agent)
├── bun.lock                       # Lockfile
└── README.md                      # Usage and configuration guide
```

The `.github/workflows/nanoclaw-agent.yml` workflow file lives at the repository root level alongside the existing GMI workflow.

---

## Configuration

### Required GitHub Secrets

| Secret | Purpose | Required |
|---|---|---|
| `ANTHROPIC_API_KEY` | Claude API authentication | Yes (unless using OAuth) |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code OAuth (alternative to API key) | Optional |

`GITHUB_TOKEN` is automatically provided by GitHub Actions with the permissions specified in the workflow.

### Environment Variables

| Variable | Default | Purpose |
|---|---|---|
| `ASSISTANT_NAME` | `Andy` | Name used in responses and memory context |
| `NANOCLAW_MAX_COMMENT_LENGTH` | `60000` | Maximum characters in an issue comment reply |
| `NANOCLAW_TASK_CHECK_INTERVAL` | `*/5 * * * *` | Cron schedule for checking due tasks |

### Workflow Inputs

The workflow can accept optional `workflow_dispatch` inputs for manual triggering and testing:

```yaml
on:
  workflow_dispatch:
    inputs:
      issue_number:
        description: 'Issue number to process'
        required: true
        type: number
      prompt:
        description: 'Override prompt (optional)'
        required: false
        type: string
```

---

## Implementation Phases

### Phase 1 — Core Agent Loop

**Goal:** A single issue triggers the agent, which responds with a comment.

- [ ] Create `.githubification/` folder structure
- [ ] Write `lifecycle/indicator.ts` — add 🚀 reaction on workflow start
- [ ] Write `lifecycle/agent.ts` — read issue, invoke agent, post reply
- [ ] Create `package.json` with `pi-coding-agent` dependency
- [ ] Write `.github/workflows/nanoclaw-agent.yml`
- [ ] Initialize `state/` directories
- [ ] Implement authorization check (collaborator permissions)
- [ ] Implement push-with-retry for state commits

### Phase 2 — Session Continuity

**Goal:** Multi-turn conversations across multiple comments on the same issue.

- [ ] Implement issue-to-session mapping in `state/issues/<N>.json`
- [ ] Pass `--session` flag to agent for conversation resume
- [ ] Commit session transcripts to `state/sessions/`
- [ ] Handle session compaction (automatic via Claude Agent SDK)

### Phase 3 — Memory System

**Goal:** Per-issue and global memory that persists across sessions.

- [ ] Create `state/groups/global/CLAUDE.md` for shared memory
- [ ] Create per-issue group folders (`state/groups/github_issue-<N>/`)
- [ ] Configure agent working directory for memory hierarchy
- [ ] Implement admin-only global memory writes (permission check)

### Phase 4 — Scheduled Tasks

**Goal:** Users can schedule recurring tasks via issue comments.

- [ ] Add `schedule` trigger to workflow
- [ ] Implement `scheduled-tasks.json` state management
- [ ] Write task scheduler logic in lifecycle script
- [ ] Post task results as issue comments
- [ ] Support cron, interval, and one-time schedules

### Phase 5 — GitHub Issues Channel Adapter

**Goal:** NanoClaw recognizes GitHub Issues as a first-class channel.

- [ ] Implement `src/channels/github-issues.ts` channel adapter
- [ ] Register channel in `src/channels/index.ts`
- [ ] Support JID format `gh:<issue_number>`
- [ ] Enable dual-mode operation (local channels + GitHub Issues)

### Phase 6 — Skills and Polish

**Goal:** Package Githubification as a NanoClaw skill.

- [ ] Create `.claude/skills/githubify/SKILL.md`
- [ ] Integrate with skills engine state tracking
- [ ] Write user documentation in `.githubification/README.md`
- [ ] Add outcome reactions (👍/👎) for success/failure indication
- [ ] Implement label-based issue filtering (optional)

---

## Design Decisions

### Why Not Run NanoClaw's Full Process on Actions?

NanoClaw's `src/index.ts` is a persistent process with a polling loop, channel connections, and container management. Running the full process on GitHub Actions would mean:

1. Starting WhatsApp/Telegram connections on each workflow run (pointless — no messages to receive)
2. Spawning Docker containers inside the Actions runner (unnecessary complexity)
3. Running the polling loop until timeout (wasteful)

Instead, v2 extracts the essential pipeline — receive message, format prompt, run agent, send response — and executes it as a single-shot lifecycle script. This is more efficient, simpler, and matches the ephemeral execution model of GitHub Actions.

### Why Git for State Instead of SQLite?

SQLite requires a persistent filesystem. GitHub Actions runners are ephemeral — the filesystem is destroyed after each run. Options considered:

1. **SQLite committed to Git** — binary diffs are opaque, merge conflicts are unresolvable
2. **SQLite as a workflow artifact** — 90-day retention limit, no cross-run sharing
3. **JSON/Markdown files in Git** — human-readable, mergeable, auditable, persistent

JSON files in Git provide the best balance of persistence, readability, and compatibility with the ephemeral runner model.

### Why Per-Issue Concurrency Groups?

NanoClaw locally serializes messages per group via `GroupQueue`. In v2, each issue maps to a NanoClaw "group." GitHub Actions concurrency groups ensure only one agent run per issue at a time, preventing:

- Race conditions on session state files
- Duplicate responses to rapidly-posted comments
- Conflicting git pushes for the same issue's state

### Why `cancel-in-progress: false`?

When a user posts a comment while the agent is already running for that issue, the new comment should be queued — not cancel the in-progress run. This matches NanoClaw's local behavior where `GroupQueue` queues messages while the container is active and processes them when it completes.

---

## Compatibility Notes

### Preserving the Local Execution Path

NanoClaw v2 does **not** remove or modify the local execution path. The same codebase supports both modes:

- **Local:** `npm run dev` starts the persistent process with channel polling and container isolation
- **GitHub:** Issue events trigger the workflow, which runs the lifecycle scripts

The GitHub Issues channel adapter (`src/channels/github-issues.ts`) returns `null` from its factory when not running in GitHub Actions, so it has zero impact on local operation.

### Version Alignment

NanoClaw v2 targets the v1.2.1 codebase. Key compatibility points:

- Channel system is skill-based (WhatsApp removed from core in v1.2.0)
- Skills engine supports three-way merge and state tracking
- Container runner supports streaming output via IPC
- Session management uses SQLite with per-group isolation

### Migration from v1

Users running NanoClaw locally can adopt v2 by:

1. Running `/githubify` skill (or manually creating the `.githubification/` folder)
2. Configuring `ANTHROPIC_API_KEY` as a GitHub Secret
3. Pushing to GitHub — the workflow activates automatically on issue events

No changes to the local installation are required. Both modes coexist.
