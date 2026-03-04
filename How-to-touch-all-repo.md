# How to Touch All Repos

A definitive guide to safely controlling many GitHub repositories from one control repository.

---

## 0) What “cross GitHub control” means

You want one repository (controller) to:

1. Discover all repos in an org/user (e.g., `japer-technology/*`)
2. Create/update files (especially `.github/workflows/*.yml`) in each repo
3. Open or update PRs in each repo
4. Optionally enable/adjust Actions settings
5. Optionally trigger workflows in target repos
6. Repeat this safely, idempotently, and audibly

This document gives the exact model to do that.

---

## 1) Recommended control model (authoritative)

Use a **GitHub App** installed on all target repos, and run a **matrix workflow** in one control repo.

### Why this is the correct model

- Least privilege per permission domain
- Revocable at org/repo level
- Scoped installation token (short-lived)
- Better auditability than PAT sprawl
- Works naturally with GitHub Actions automation

> Avoid using personal tokens for fleet-wide automation unless you are in emergency mode.

---

## 2) Required permissions (exact)

Configure your GitHub App with these **Repository permissions**:

- **Metadata: Read-only** (baseline)
- **Contents: Read and write** (file/branch changes)
- **Pull requests: Read and write** (PR create/update)
- **Actions: Read and write** (workflow dispatch / actions operations)
- **Workflows: Read and write** (required to modify `.github/workflows/*`)

Optional:
- **Administration: Read and write** if your bot must change repo-level Actions settings or branch policies.

No org permissions are required for basic PR/file sync across installed repos.

---

## 3) Create and install the GitHub App

1. Go to **GitHub → Settings → Developer settings → GitHub Apps → New GitHub App**
2. Name it (e.g., `gitification-bot`)
3. Set repository permissions listed above
4. Create app
5. Generate and download **private key** (`.pem`)
6. Note **App ID**
7. Install app onto `japer-technology` and select **All repositories**

Result: app can mint per-installation access tokens.

---

## 4) Store credentials in the control repo

In the control repo (e.g., `japer-technology/githubification`) add Actions secrets:

- `GITIFICATION_APP_ID` → numeric App ID
- `GITIFICATION_APP_PRIVATE_KEY` → full PEM text

Do **not** commit keys to git.

---

## 5) Control repo structure

Recommended:

```text
.github/
  workflows/
    fleet-sync.yml
scripts/
  apply-repo-change.sh
templates/
  workflows/
    github-minimum-intelligence-agent.yml
manifests/
  repos.json (optional static allowlist)
```

Design rule: templates live in control repo; targets get synchronized via PR.

---

## 6) Fleet workflow (matrix orchestrator)

Create `.github/workflows/fleet-sync.yml`:

```yaml
name: fleet-sync

on:
  workflow_dispatch:
    inputs:
      dry_run:
        description: "Do not push/create PR"
        required: false
        default: "false"
  schedule:
    - cron: "17 3 * * *"

permissions:
  contents: read

jobs:
  discover:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.mk.outputs.matrix }}
    steps:
      - name: Mint org app token
        id: app
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.GITIFICATION_APP_ID }}
          private-key: ${{ secrets.GITIFICATION_APP_PRIVATE_KEY }}
          owner: japer-technology

      - name: Build repo matrix
        id: mk
        env:
          GH_TOKEN: ${{ steps.app.outputs.token }}
        run: |
          gh api "/orgs/japer-technology/repos?per_page=100&type=all" --paginate \
            --jq '[ .[] | select(.archived|not) | select(.disabled|not) | .name ]' > repos.json
          echo "matrix={\"repo\":$(cat repos.json)}" >> "$GITHUB_OUTPUT"

  apply:
    needs: discover
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      max-parallel: 5
      matrix: ${{ fromJson(needs.discover.outputs.matrix) }}

    steps:
      - uses: actions/checkout@v4

      - name: Mint per-repo app token
        id: app
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.GITIFICATION_APP_ID }}
          private-key: ${{ secrets.GITIFICATION_APP_PRIVATE_KEY }}
          owner: japer-technology
          repositories: ${{ matrix.repo }}

      - name: Apply change to repo
        env:
          GH_TOKEN: ${{ steps.app.outputs.token }}
          ORG: japer-technology
          REPO: ${{ matrix.repo }}
          DRY_RUN: ${{ github.event.inputs.dry_run || 'false' }}
        run: bash scripts/apply-repo-change.sh
```

---

## 7) Per-repo apply script (idempotent)

Create `scripts/apply-repo-change.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${GH_TOKEN:?GH_TOKEN required}"
: "${ORG:?ORG required}"
: "${REPO:?REPO required}"
DRY_RUN="${DRY_RUN:-false}"

tmp="$(mktemp -d)"
trap 'rm -rf "$tmp"' EXIT

repo_url="https://x-access-token:${GH_TOKEN}@github.com/${ORG}/${REPO}.git"
git clone --quiet "$repo_url" "$tmp/repo"
cd "$tmp/repo"

default_branch="$(gh api "repos/${ORG}/${REPO}" --jq '.default_branch')"
branch="gitification/managed-workflow-sync"

git checkout -B "$branch" "origin/${default_branch}"

mkdir -p .github/workflows
cp "${GITHUB_WORKSPACE}/templates/workflows/github-minimum-intelligence-agent.yml" \
   .github/workflows/github-minimum-intelligence-agent.yml

if git diff --quiet; then
  echo "[${ORG}/${REPO}] no change"
  exit 0
fi

if [[ "$DRY_RUN" == "true" ]]; then
  echo "[${ORG}/${REPO}] DRY RUN: changes detected, skipping push/PR"
  exit 0
fi

git config user.name "gitification-bot[app]"
git config user.email "gitification-bot[app]@users.noreply.github.com"

git add .github/workflows/github-minimum-intelligence-agent.yml
git commit -m "chore(gitification): sync managed workflow"
git push --force origin "$branch"

existing_pr="$(gh pr list --repo "${ORG}/${REPO}" --head "$branch" --json number --jq '.[0].number // empty')"

if [[ -n "$existing_pr" ]]; then
  gh pr edit "$existing_pr" --repo "${ORG}/${REPO}" \
    --title "chore(gitification): sync managed workflow" \
    --body "Automated by ${GITHUB_REPOSITORY}"
else
  gh pr create --repo "${ORG}/${REPO}" \
    --base "$default_branch" --head "$branch" \
    --title "chore(gitification): sync managed workflow" \
    --body "Automated by ${GITHUB_REPOSITORY}"
fi
```

Make it executable:

```bash
chmod +x scripts/apply-repo-change.sh
```

---

## 8) Activating workflows in target repos

A workflow is active once:

1. The workflow YAML exists on the repo’s default branch
2. GitHub Actions is enabled for that repo
3. Org/repo Actions policy allows that workflow/actions usage

If you need bot-driven settings changes (requires `Administration: write`):

```bash
gh api -X PUT "repos/${ORG}/${REPO}/actions/permissions" \
  -f enabled=true -f allowed_actions=all

gh api -X PUT "repos/${ORG}/${REPO}/actions/permissions/workflow" \
  -f default_workflow_permissions=write \
  -F can_approve_pull_request_reviews=true
```

---

## 9) Operational controls you should enforce

1. **PR-only writes** to default branch (never direct push)
2. **Fixed branch naming** per sync type (`gitification/...`)
3. **Concurrency cap** (`max-parallel: 3-10`) to avoid abuse/rate spikes
4. **Allowlist/denylist** for critical repos
5. **Dry-run mode** before rollout
6. **Signed provenance** in commit/PR body with controller run URL
7. **Manual approval** for sensitive repos

---

## 10) Security and governance checklist

- [ ] GitHub App private key only in Actions secrets
- [ ] Least privilege permissions only
- [ ] App installed only where needed
- [ ] Branch protection requires checks/review before merge
- [ ] PR templates explain automated origin
- [ ] Audit logs reviewed periodically
- [ ] Emergency kill switch (disable workflow or uninstall app)

---

## 11) Troubleshooting (common failures)

### “Resource not accessible by integration”
Cause: missing app permission or app not installed on target repo.
Fix: add permission + reinstall/expand installation.

### Cannot modify `.github/workflows/*`
Cause: no **Workflows: write** permission.
Fix: grant **Workflows read/write** on app.

### PR creation fails
Cause: missing `pull_requests: write` or branch protections conflict.
Fix: grant permission; ensure bot can push feature branch.

### Workflow present but not running
Cause: Actions disabled/policy blocked/not on default branch.
Fix: enable Actions; merge file to default branch.

---

## 12) Scale limits and practical notes

- Use pagination (`--paginate`) for orgs with many repos
- Keep `max-parallel` moderate (5 is a good start)
- Consider shard runs by repo prefix for very large fleets
- Keep templates versioned; include changelog in PR bodies

---

## 13) Minimal rollout plan (recommended)

1. Pilot on 2–3 low-risk repos
2. Validate PR quality and workflow behavior
3. Expand to 10 repos
4. Expand to all repos with scheduled runs
5. Add policy checks and drift detection

---

## 14) Emergency rollback

If something goes wrong:

1. Disable `fleet-sync` workflow in controller repo
2. Uninstall GitHub App (or remove affected repos from installation)
3. Close open bot PRs in target repos
4. Revert merged commits using generated PR list

---

## 15) PAT fallback (not preferred)

If App is impossible temporarily, use a fine-grained PAT with per-repo access and permissions:
- Contents read/write
- Pull requests read/write
- Actions read/write
- Workflows read/write

This works, but is less secure and less maintainable than GitHub App.

---

## Final recommendation

For `japer-technology/*`, the definitive production approach is:

- **GitHub App** with precise permissions
- **One controller repo** (`githubification`)
- **Matrix orchestrator workflow**
- **Idempotent per-repo apply script**
- **PR-based rollout with branch protections**

That gives safe, auditable, repeatable cross-repo control.
