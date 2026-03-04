# NanoClaw v3 Specification

### Githubification of `japer-technology/github-nanoclaw` (Type 1 — AI Agent Repo)

---

## Status

**This v3 spec is the reconciled implementation spec** after deep comparison of:

- `/.githubification/specification/nanoclaw.md`
- `/.githubification/specification/nanoclaw-v2.md`
- NanoClaw source architecture (`src/index.ts`, `src/db.ts`, `src/container-runner.ts`, `src/group-queue.ts`, `src/ipc.ts`, `src/task-scheduler.ts`, `container/agent-runner/*`)
- Githubification lessons (NanoClaw, Claw, GMI, MicroClaw, IronClaw, consolidated playbook)

v3 resolves contradictions between v1-style minimalism and v2-style redesign by choosing a **compatibility-first, incremental channel-addition architecture**.

---

## 1) Deep Analysis — What v1 and v2 got right/wrong

### 1.1 Core conflict map

| Topic | `nanoclaw.md` | `nanoclaw-v2.md` | v3 decision |
|---|---|---|---|
| Folder boundary | `.github-nanoclaw/` self-contained | `.githubification/` runtime folder | **`.github-nanoclaw/`** for deployable product boundary |
| Runtime execution | Keep containerized execution | Prefer direct runner, no nested container | **Container-first default**, runner-direct optional later |
| State model | Keep SQLite + readable exports | Replace SQLite with JSON-first state | **SQLite as canonical + required readable exports** |
| Scheduler | Out of scope for v1 | Included in main design | **Out of v3.0 core; separate phase/workflow** |
| Strategy claim | Channel addition + one-shot | Channel addition but with larger rewrites | **True channel-addition with minimal invasive one-shot mode** |
| Risk profile | Low-medium change | Medium-high redesign | **Low-medium for v3.0; high-risk ideas moved to later phases** |

### 1.2 Architectural truths from source code

1. NanoClaw is already strongly modular where it matters for Githubification:
   - channel abstraction (`src/channels/registry.ts`)
   - transport-independent message model (`NewMessage`)
   - isolated execution (`runContainerAgent`)
2. Biggest mismatch is lifecycle shape:
   - NanoClaw local: daemon + polling + multi-channel
   - GitHub Actions: event-driven, ephemeral, one-shot
3. Rewriting persistence from SQLite to JSON at the same time as runtime migration is unnecessary risk.
4. Security posture in NanoClaw is built around container isolation + mount policy; dropping that at v3.0 reduces parity with the source system.

### 1.3 v3 principle

> **Preserve NanoClaw’s proven internals; adapt only ingress/egress and lifecycle to GitHub’s event model.**

---

## 2) Strategy and scope

### 2.1 Primary strategy

**Strategy 5 (Channel Addition) + one-shot GitHub execution mode**.

GitHub Issues become a first-class NanoClaw channel surface. The agent run model is one-shot per event, not a persistent daemon.

### 2.2 v3.0 in scope

1. Issue/comment-triggered execution.
2. Per-issue session continuity.
3. Containerized agent execution on Actions.
4. Git-committed state persistence.
5. Required readable state exports.
6. Fail-closed authorization + sentinel controls.
7. Per-issue concurrency and resilient push retry.

### 2.3 v3.0 out of scope

1. Production scheduler parity (`schedule_task`) as part of same workflow.
2. Full multi-channel runtime activation on Actions.
3. Skill-engine integration into installer path.
4. Public anonymous usage.

---

## 3) Non-negotiable invariants

1. **Self-contained install unit:** `.github-nanoclaw/`
2. **Fail-closed startup:** auth + sentinel + event validation must pass before agent runs.
3. **Issue isolation:** no session/memory leakage across issues.
4. **Security parity:** keep container boundary in v3.0.
5. **Git as durable state:** every run commits state changes.
6. **Auditability:** binary DB must have human-readable companion exports.
7. **No host repo lock-in:** install/remove should not require invasive host changes.

---

## 4) Target architecture

### 4.1 Primitive mapping

| GitHub Primitive | NanoClaw v3 mapping |
|---|---|
| **Actions** | Ephemeral compute per issue event |
| **Git** | Durable state + memory + session artifacts |
| **Issues** | Conversation UI (one issue = one thread/group) |
| **Secrets** | Credential source for Claude and GitHub API |

### 4.2 Execution pipeline

```
Issue open/comment event
  -> Auth + guard checks
  -> Indicator start (🚀)
  -> Build NanoClaw message context for issue N
  -> One-shot agent execution (containerized)
  -> Post response comment
  -> Export readable state
  -> Commit/push with retry
  -> Indicator end (👍 success / 👎 failure)
```

### 4.3 Group identity mapping

- `chat_jid`: `gh:<owner>/<repo>#<issue_number>`
- `group.folder`: `github-issue-<issue_number>`
- `isMain`: false for issue groups
- optional admin-control issue/group can be added later

---

## 5) File and folder specification

### 5.1 Githubified product folder

```
.github-nanoclaw/
├── README.md
├── AGENTS.md
├── github-nanoclaw-ENABLED.md
├── lifecycle/
│   ├── authorize.ts
│   ├── indicator.ts
│   ├── github-agent.ts
│   ├── export-readable-state.ts
│   └── commit-state.ts
├── install/
│   ├── github-nanoclaw-INSTALLER.ts
│   ├── github-nanoclaw-WORKFLOW.yml
│   ├── github-nanoclaw-ISSUE-TEMPLATE.md
│   └── github-nanoclaw-AGENTS.md
├── state/
│   ├── store/messages.db
│   ├── groups/
│   │   ├── global/CLAUDE.md
│   │   └── github-issue-<n>/CLAUDE.md
│   ├── data/
│   │   ├── sessions/
│   │   └── ipc/
│   ├── issues/
│   │   └── <n>.json
│   └── readable/
│       ├── issues/<n>.md
│       ├── sessions-index.json
│       └── latest-run.json
└── tests/
    ├── mapping.test.ts
    ├── auth-guard.test.ts
    ├── isolation.test.ts
    ├── state-export.test.ts
    ├── commit-retry.test.ts
    ├── workflow-structure.test.ts
    └── config-overrides.test.ts
```

### 5.2 Host repo touch points (only)

Installer may create/update:

- `.github/workflows/github-nanoclaw-agent.yml`
- `.github/ISSUE_TEMPLATE/github-nanoclaw-chat.md`

No other host-level mutation required.

---

## 6) Code-level changes required in `github-nanoclaw`

### 6.1 Add GitHub one-shot orchestrator

Create `src/github-agent.ts` (or `src/modes/github.ts`) that:

1. reads `GITHUB_EVENT_PATH`
2. determines event kind (`issues.opened`, `issues.reopened`, `issue_comment.created`)
3. normalizes inbound message(s) to `NewMessage`
4. initializes DB and registered issue-group if absent
5. executes a single run via `runContainerAgent`
6. collects outbound messages for the lifecycle layer to post via the outbound adapter (section 6.4)

### 6.2 Add path overrides in config

`src/config.ts` must support env overrides:

- `NANOCLAW_STORE_DIR`
- `NANOCLAW_GROUPS_DIR`
- `NANOCLAW_DATA_DIR`

GitHub workflow sets these to `.github-nanoclaw/state/{store,groups,data}`.

### 6.3 Add GitHub mode flags

Introduce runtime flags:

- `NANOCLAW_GITHUB_MODE=1`
- `NANOCLAW_ENABLE_SCHEDULER=0` (default in issue workflow)
- `NANOCLAW_ENABLE_CHANNEL_BOOT=0` (skip normal channel connect loop)

### 6.4 Add GitHub outbound adapter

Provide posting adapter (either via tiny GitHub channel or lifecycle helper):

- post issue comment(s)
- add/remove status reactions
- strip `<internal>...</internal>` from outbound text

### 6.5 Keep container boundary for v3.0

`runContainerAgent` remains primary execution path in Actions.

GitHub mode sets:

- short idle timeout (e.g. 10–20s) to avoid waiting 30m default
- bounded container timeout suitable for workflow constraints

---

## 7) Workflow specification

### 7.1 Triggers

```yaml
on:
  issues:
    types: [opened, reopened]
  issue_comment:
    types: [created]
```

(`edited` is excluded in v3.0 to avoid replay/duplicate semantics; optional later.)

### 7.2 Concurrency

```yaml
concurrency:
  group: github-nanoclaw-${{ github.repository }}-issue-${{ github.event.issue.number }}
  cancel-in-progress: false
```

### 7.3 Required pipeline

1. **Authorize** actor permission (`admin|maintain|write`)
2. **Guard** sentinel file exists
3. **Indicator start** (🚀)
4. **Run one-shot agent**
5. **Post response comment**
6. **Export readable state**
7. **Commit and push with retry**
8. **Indicator end** (👍 success / 👎 failure)

### 7.4 Push retry policy

- max 10 attempts
- backoff escalation (1s → 3s → 5s ...)
- conflict handling with `git pull --rebase -X theirs`
- fail with actionable error after final attempt

---

## 8) State model

### 8.1 Canonical persistence

SQLite remains canonical in v3.0:

- `state/store/messages.db`

This preserves compatibility with existing NanoClaw code and minimizes migration risk.

### 8.2 Required readable mirrors

After each run, export:

1. issue summary markdown (`state/readable/issues/<n>.md`)
2. session index (`state/readable/sessions-index.json`)
3. latest run metadata (`state/readable/latest-run.json`)
4. issue mapping (`state/issues/<n>.json`)

### 8.3 Canonical mapping object

`state/issues/<n>.json`:

```json
{
  "issueNumber": 42,
  "chatJid": "gh:owner/repo#42",
  "groupFolder": "github-issue-42",
  "sessionId": "...",
  "sessionPath": "state/data/sessions/github-issue-42/.claude/...",
  "lastProcessedAt": "2026-03-04T00:00:00Z"
}
```

---

## 9) Security model

### 9.1 Fail-closed gates

All must pass before execution:

1. actor authorization
2. sentinel existence
3. event type validation
4. bot-loop prevention

### 9.2 Credential handling

- only required env vars injected
- no secrets written into committed files
- reuse NanoClaw filtering model for allowed auth variables

### 9.3 Isolation model

v3.0 uses layered isolation:

1. GitHub Actions ephemeral runner boundary
2. NanoClaw container execution boundary
3. mount restrictions + blocked patterns

### 9.4 Loop prevention

Ignore event when:

- actor login ends with `[bot]`
- actor equals configured bot account
- payload originated from agent-authored comment

---

## 10) UX and behavior

### 10.1 Reactions

- 🚀 when processing starts
- 👍 when complete
- 👎 on auth failure or runtime error

### 10.2 Response policy

v3.0 default:

- one consolidated issue reply per run
- no internal reasoning tags
- include brief failure reason when no model output

### 10.3 Trigger semantics

In GitHub mode, comment creation is itself the trigger; explicit `@Assistant` prefix is not required.

---

## 11) Installer specification

Installer requirements:

1. idempotent file creation
2. never clobber user-customized files silently
3. initialize clean state
4. create workflow + issue template + `.github-nanoclaw/` skeleton
5. verify required secrets are documented

Uninstall requirements:

- remove workflow/template entries created by installer
- optionally retain or remove `.github-nanoclaw/state/` by user choice

---

## 12) Testing specification

### 12.1 Unit/structural tests (required)

1. event→issue mapping correctness
2. issue isolation guarantees
3. auth and sentinel gate behavior
4. export-readable-state correctness
5. commit retry loop behavior
6. workflow schema/structure checks
7. config path override correctness

### 12.2 Integration tests (required before release)

1. simulate `issues.opened`
2. simulate `issue_comment.created`
3. verify resume on second comment same issue
4. verify no leakage between issue A and issue B

### 12.3 Non-goal tests

- do not assert exact LLM wording

---

## 13) Phased implementation plan

### Phase 0 — Foundations

- add config path overrides
- add GitHub mode flags
- add one-shot orchestrator skeleton
- add minimal workflow + auth + sentinel

### Phase 1 — Functional core (v3.0)

- issue/comment execution path
- containerized one-shot run
- issue reply posting
- state commit + retry
- readable export generation

### Phase 2 — Hardening

- richer error taxonomy + operator diagnostics
- stricter loop detection
- improved audit exports
- expanded integration fixtures

### Phase 3 — Optional enhancements

- runner-direct execution mode (opt-in)
- `issue_comment.edited` handling semantics
- scheduler workflow (separate `on: schedule` pipeline)
- full channel registry integration for GitHub channel class

---

## 14) Acceptance criteria (release gate)

v3.0 is accepted only if all pass:

1. issue open triggers response on Actions
2. new comment on same issue resumes context
3. different issues remain isolated
4. unauthorized actor cannot execute agent
5. sentinel removal stops all processing
6. state persists across runs through git commits
7. push conflicts resolve via retry/rebase loop
8. readable exports are updated each run
9. no secrets appear in committed state
10. workflow runtime is bounded and does not hang on idle timeout defaults

---

## 15) Final decision summary

1. **Canonical deployable folder:** `.github-nanoclaw/`
2. **Execution model:** one-shot GitHub mode, not daemon
3. **Security baseline:** container-first in v3.0
4. **Persistence baseline:** SQLite canonical + mandatory readable mirrors
5. **Scope control:** scheduler deferred to dedicated later phase
6. **Strategy fidelity:** true channel addition with minimal invasive adaptation

---

## 16) Implementation checklist (task-ready)

1. Add config env override support in `src/config.ts`.
2. Create `src/github-agent.ts` one-shot orchestrator.
3. Add GitHub mode gate logic to skip scheduler/channel boot loops.
4. Add lifecycle scripts in `.github-nanoclaw/lifecycle/`.
5. Add workflow template and installer.
6. Implement readable export script.
7. Add commit retry module.
8. Add tests (unit + integration fixtures).
9. Validate on test repo with parallel issue traffic.
10. Tag release once acceptance criteria pass.
