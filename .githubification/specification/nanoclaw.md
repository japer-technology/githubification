# NanoClaw Githubification Specification

## Purpose

Define a concrete, implementation-ready specification for Githubifying [`japer-technology/github-nanoclaw`](https://github.com/japer-technology/github-nanoclaw) using the Githubification patterns in this repo.

This specification is based on deep review of:

- `.githubification/README.md`
- `.githubification/lesson-consolidation.md`
- `.githubification/lesson-from-nanoclaw.md`
- `.githubification/lesson-from-claw.md`
- `.githubification/lesson-from-gmi.md`
- `.githubification/lesson-from-microclaw.md`
- `.githubification/lesson-from-ironclaw.md`
- `github-nanoclaw` source and docs (`README.md`, `docs/SPEC.md`, `docs/SECURITY.md`, `docs/REQUIREMENTS.md`, `src/*`, `container/*`, `skills-engine/*`)

---

## 1) Strategy Selection

### Selected strategy: **Channel Addition + GitHub one-shot execution mode**

NanoClaw already has a channel-agnostic architecture (`src/channels/registry.ts` + channel factory pattern), containerized execution, and persistent session/memory model.

From the consolidated Githubification playbook, this maps most strongly to:

- **Strategy 5 — Channel Addition** (MicroClaw-style)
- with a NanoClaw-specific runtime adaptation: **ephemeral one-shot mode for Actions**

### Why this strategy

- NanoClaw is already multi-channel and self-registering.
- Core orchestration (`runContainerAgent`, group queue, session persistence) already exists.
- Security model (container isolation + mount controls + credential filtering) is already stronger than most wrappers.
- The largest mismatch is not architecture, but runtime shape: NanoClaw is daemon/polling; GitHub Actions is event-driven/ephemeral.

---

## 2) Scope

## In scope (v1)

1. Run NanoClaw from GitHub issue/comment events.
2. Persist conversation state across workflow runs via git-committed state.
3. Keep per-issue memory/session isolation.
4. Keep fail-closed execution and explicit authorization.
5. Keep containerized agent execution in Actions.
6. Post agent responses back to the same issue.
7. Commit state updates with robust git retry/rebase logic.

## Out of scope (v1)

1. Long-running scheduler tasks (`schedule_task`) in GitHub mode.
2. Multi-channel runtime in GitHub mode (WhatsApp/Telegram/etc).
3. Full skill-engine migration into Githubification installer flow.
4. Public unauthenticated usage by default.

---

## 3) Target Architecture

## 3.1 GitHub primitive mapping

| GitHub Primitive | NanoClaw mapping in GitHub mode |
|---|---|
| **Actions** | Compute runtime per issue event |
| **Git** | Persistent state (`.github-nanoclaw/state/*`) |
| **Issues** | Chat UI (one issue = one conversation thread) |
| **Secrets** | Claude/auth credentials + GitHub token |

## 3.2 Runtime model

GitHub mode is **single-event, single-run orchestration**:

1. Workflow receives issue/comment event.
2. Authorization/guard checks run first.
3. Event converted to NanoClaw inbound message(s).
4. Agent executes once for that issue context.
5. Response is posted to issue.
6. State is committed and pushed.

No always-on polling daemon is required in GitHub mode.

## 3.3 Group model in GitHub mode

- `chat_jid` format: `gh:<owner>/<repo>#<issue_number>`
- Group folder: `github-issue-<issue_number>`
- Every issue is isolated by default.
- Optional global memory file remains readable, but writes are maintainers-only.

---

## 4) Files and Folder Layout

Add a self-contained Githubification folder:

```
.github-nanoclaw/
├── README.md
├── AGENTS.md
├── github-nanoclaw-ENABLED.md
├── lifecycle/
│   ├── authorize.ts
│   ├── indicator.ts
│   ├── github-agent.ts
│   └── commit-state.ts
├── install/
│   ├── github-nanoclaw-INSTALLER.ts
│   ├── github-nanoclaw-workflow.yml
│   ├── github-nanoclaw-issue-template.md
│   └── github-nanoclaw-AGENTS.md
├── state/
│   ├── messages.db
│   ├── groups/
│   │   ├── global/CLAUDE.md
│   │   └── github-issue-<n>/CLAUDE.md
│   ├── sessions/
│   ├── ipc/
│   └── readable/
│       ├── issues/<n>.md
│       └── sessions-index.json
└── tests/
    ├── github-agent.test.ts
    ├── mapping.test.ts
    └── workflow-structure.test.ts
```

Notes:
- This follows the self-contained folder invariant from the consolidated playbook.
- Existing NanoClaw root code remains source of execution logic; `.github-nanoclaw/` contains GitHub runtime orchestration, state, templates, and docs.

---

## 5) Required Code Changes in `github-nanoclaw`

## 5.1 Add explicit GitHub mode entrypoint

Create `src/github-agent.ts` (or equivalent) that:

1. Reads event payload from `GITHUB_EVENT_PATH`.
2. Maps issue/comment into `NewMessage` records.
3. Initializes DB + registered group for issue.
4. Runs a single message-processing cycle for that issue.
5. Returns final outbound text for posting.

## 5.2 Make paths overridable for GitHub mode

`src/config.ts` currently derives directories from `process.cwd()`. Add env overrides:

- `NANOCLAW_STORE_DIR`
- `NANOCLAW_GROUPS_DIR`
- `NANOCLAW_DATA_DIR`

In GitHub mode, set these to `.github-nanoclaw/state/...`.

## 5.3 Disable unsupported capabilities in GitHub mode

In v1 GitHub mode:

- Disable scheduler loop startup.
- Disable channel connect loop.
- Disable task IPC operations that depend on persistent daemon semantics.

Agent execution remains fully containerized via existing `runContainerAgent`.

## 5.4 Add GitHub outbound adapter

Implement minimal outbound target:

- Post final agent output as issue comment (GitHub REST API / `gh api`).
- Preserve internal tag stripping (`<internal>...</internal>`).

---

## 6) Workflow Specification

Workflow trigger:

- `issues: [opened, reopened]`
- `issue_comment: [created, edited]`

Ignore events when:

- actor is bot/self
- comment is command excluded by policy
- sentinel file missing

Concurrency:

```yaml
concurrency:
  group: github-nanoclaw-${{ github.repository }}-issue-${{ github.event.issue.number }}
  cancel-in-progress: false
```

Pipeline:

1. **Authorize** (maintain/write/admin)
2. **Guard** (`.github-nanoclaw/github-nanoclaw-ENABLED.md` exists)
3. **Indicator Start** (🚀 reaction)
4. **Run Agent** (`github-agent.ts`)
5. **Commit/Push State** (retry loop)
6. **Indicator End** (👍 success, 👎 failure)

Commit push retry requirements:

- up to 10 attempts
- exponential or stepped backoff
- `git pull --rebase -X theirs` on conflict

---

## 7) Security Specification

## 7.1 Fail-closed controls

Must pass all before agent executes:

1. Sentinel file exists.
2. Actor permission authorized.
3. Event source is valid issue/comment event.

## 7.2 Secret handling

- Reuse NanoClaw credential filtering model.
- Only pass required auth env vars to container input.
- Never write secrets into committed state.

## 7.3 Isolation guarantees

- Keep containerized agent execution.
- Keep per-issue folder/session isolation.
- Keep mount validation and blocked pattern checks.

## 7.4 Loop prevention

- Ignore comments authored by the agent bot account.
- Ignore events generated by state-commit pushes.

---

## 8) State and Memory Specification

## 8.1 Canonical mapping

```
Issue #N
  -> chat_jid gh:<repo>#N
  -> registered_groups row
  -> folder github-issue-N
  -> sessions/github-issue-N/.claude/
```

## 8.2 Persistence policy

Commit after each successful run:

- SQLite DB (`messages.db`)
- group memory files
- session artifacts needed for resume
- readable exports for auditability

Readable exports are required because SQLite diffs are opaque.

## 8.3 Memory permissions

- Issue-local memory: writable in issue context.
- Global memory writes: maintainers only (or disabled by default in v1).

---

## 9) UX Specification

## Required reactions

- 🚀 when run starts
- 👍 when run succeeds
- 👎 when unauthorized/failure

## Required response behavior

- One coherent issue reply per run (v1 default).
- Strip internal reasoning tags.
- Include actionable status when run fails.

---

## 10) Installer Specification

Installer must:

1. Create `.github/workflows/github-nanoclaw-agent.yml`.
2. Create `.github/ISSUE_TEMPLATE/github-nanoclaw-chat.md`.
3. Initialize `.github-nanoclaw/` if absent.
4. Be idempotent (never overwrite user-customized files without opt-in).
5. Initialize clean state (no inherited sessions/persona from source repo).

---

## 11) Testing Specification

Test structure, not model output.

Required tests:

1. Event-to-group mapping correctness.
2. Issue isolation (no cross-issue session leakage).
3. Authorization gate behavior.
4. Sentinel gate behavior.
5. Commit retry loop behavior.
6. Workflow file validity.
7. GitHub mode path overrides and state placement.

Optional integration tests:

- Simulated `issues.opened` and `issue_comment.created` payload runs.

---

## 12) Rollout Plan

## Phase 0 — Foundations

- Add GitHub-mode path overrides.
- Add `src/github-agent.ts` one-shot entrypoint.
- Add workflow with auth/guard/indicator skeleton.

## Phase 1 — Functional v1

- Execute one-shot issue processing.
- Post issue comment responses.
- Commit/push state with retry.
- Add structural tests.

## Phase 2 — Hardening

- Add readable state exports.
- Improve error taxonomy and user-facing failure messages.
- Add optional maintainer control issue for global memory.

## Phase 3 — Advanced (optional)

- GitHub-native scheduled task model (cron workflows + task table bridge).
- Formal GitHub channel adapter integrated into channel registry.
- Installer polish and distribution flow.

---

## 13) Acceptance Criteria

A NanoClaw Githubification is complete when:

1. Opening/commenting on an issue triggers NanoClaw response on GitHub Actions.
2. Conversation resumes correctly on subsequent comments in same issue.
3. Different issues are isolated in memory/session state.
4. Unauthorized users cannot execute the agent.
5. State is committed and survives across runs.
6. Concurrency conflicts are handled without lost responses.
7. Disabling via sentinel file immediately halts execution.

---

## 14) Key Design Decision Summary

1. **Use Channel Addition strategy** because NanoClaw already has channel abstraction.
2. **Use one-shot GitHub runtime mode** because Actions are ephemeral.
3. **Keep container isolation** as a first-class invariant.
4. **Persist state in git with readable exports** to balance compatibility and auditability.
5. **Keep fail-closed controls** (auth + sentinel + loop prevention).
6. **Ship as self-contained `.github-nanoclaw/` unit** following Githubification composition principles.
