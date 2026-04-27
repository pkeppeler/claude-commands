---
description: Audit a GitHub repo's settings, security features, rulesets, and convention files (read-only)
---

Audit a GitHub repository's configuration. **Read-only — make no changes.**

**Target repo**: `$ARGUMENTS` in `owner/name` form. If empty, derive from the current directory using `gh repo view --json nameWithOwner --jq .nameWithOwner`. If that fails (not a git repo, no GitHub remote), ask the user.

Use `gh api` throughout. Run independent reads in parallel via multiple Bash tool uses in one message.

## Step 0 — Preflight

Verify `gh` is installed and authenticated **before** any other work.

- `gh --version` — must succeed. If not, stop and tell the user:
  - macOS: `brew install gh`
  - Linux/Windows: see https://github.com/cli/cli#installation
- `gh auth status` — must show authenticated. If not, stop and tell the user to run `gh auth login` (browser flow) or set a `GH_TOKEN` / `GITHUB_TOKEN` env var.

If either check fails, **do not proceed**. Print the remediation and exit.

## What to inventory

### 1. Repo metadata — `gh api repos/<o>/<n>`
- `visibility`, `default_branch`, `owner.type`, `owner.plan.name` (free vs paid)
- Merge settings: `allow_squash_merge`, `allow_merge_commit`, `allow_rebase_merge`, `delete_branch_on_merge`, `allow_auto_merge`, `allow_update_branch`, `squash_merge_commit_title`, `squash_merge_commit_message`
- `security_and_analysis.{secret_scanning, secret_scanning_push_protection, dependabot_security_updates}.status` (may be absent on private/free)

### 2. Vulnerability alerts — `gh api repos/<o>/<n>/vulnerability-alerts`
- 204 → enabled; 404 → disabled

### 3. Private Vulnerability Reporting — `gh api repos/<o>/<n>/private-vulnerability-reporting`
- Returns `{"enabled": true/false}` or 404 if unavailable on this plan

### 4. Rulesets — `gh api repos/<o>/<n>/rulesets`
- For each ruleset: name, target, enforcement, rules summary, bypass actors
- 403 → "not available on current plan" (private + free)
- For any ruleset whose target includes the default branch, fetch its full rule details with `gh api repos/<o>/<n>/rulesets/<id>` so individual rule parameters and bypass-actor modes are visible.

### 5. Legacy branch protection — `gh api repos/<o>/<n>/branches/<default_branch>/protection`
- 200 → list active protections
- 404 → "not set"
- 403 → "not available"

### 6. Convention files — `gh api repos/<o>/<n>/contents/<path>`
Check presence (not content) of: `.github/CODEOWNERS`, `.github/dependabot.yml`, `SECURITY.md`, `.github/PULL_REQUEST_TEMPLATE.md`, `.github/ISSUE_TEMPLATE/`, `.github/workflows/`

### 7. Workflow job names — read `.github/workflows/*.yml` via `gh api`
For each workflow, list job keys and `name:` values. These are the candidate contexts for required status checks in a ruleset.

## First-class baseline

Compare findings against this baseline (mirrors what `/bootstrap-github-repo` applies):

**Repo settings**
- `allow_squash_merge: true`, `allow_merge_commit: false`, `allow_rebase_merge: false`
- `delete_branch_on_merge: true`
- `allow_auto_merge: true`, `allow_update_branch: true`
- `squash_merge_commit_title: PR_TITLE`, `squash_merge_commit_message: BLANK`

**Security**
- Vulnerability alerts: enabled
- Dependabot security updates: enabled
- Secret scanning: enabled (public free; private requires GHAS)
- Push protection: enabled (public free; private requires GHAS)
- Private Vulnerability Reporting: enabled

**Default-branch ruleset** (named `default-branch-protection`)
- Enforcement: `active`
- `pull_request` rule: `required_approving_review_count: 1`, `dismiss_stale_reviews_on_push: true`, `require_code_owner_review: true`, `required_review_thread_resolution: true`, `allowed_merge_methods: ["squash"]` (mirrors the repo-level squash-only policy as audit-visible defense in depth)
- `required_status_checks` rule — **conditional on workflow presence**:
  - Workflows exist → rule must be present with `strict_required_status_checks_policy: true` and contexts that exactly match the discovered job names (no missing, no stale)
  - No workflows → rule must be **absent** (a placeholder always-passing check is cargo cult)
- `non_fast_forward`, `deletion`, `required_linear_history` rules present
- Bypass actor: `RepositoryRole` admin (id 5) with **`bypass_mode: "pull_request"`** (admin can self-merge a PR without review, but **must still open a PR** — `"always"` is a gap because it permits direct pushes to main)

**Files**
- `.github/CODEOWNERS` exists with at least `* @<owner-login>`
- `.github/dependabot.yml` exists (at minimum: `github-actions` ecosystem)
- `SECURITY.md` exists for public repos

**Legacy branch protection on default branch**
- Should be absent — the new ruleset replaces it. Presence (especially a no-op one) is a gap.

## Output

A single markdown report under ~500 words:

1. **Header** — repo, visibility, plan, default branch, and one-line plan summary (e.g. "private + free → no rulesets / no GHAS scanning")
2. **Settings table** — current value vs. baseline, with ✅/❌/⏭️ (skipped/unavailable) for each baseline setting above
3. **Rulesets** — list with summaries; for the default-branch ruleset, mark each baseline criterion ✅/❌, including the bypass mode
4. **Legacy branch protection** — present (and what it enforces) or "not set"
5. **Files** — which baseline files are present / missing
6. **Workflow jobs** — discovered job names. Compare against the ruleset's `required_status_checks` contexts and flag both directions:
   - Job exists in a workflow but not listed as a required context → gap (re-run bootstrap)
   - Context listed in the ruleset but no matching job in any workflow → gap (stale context, re-run bootstrap)
   - No workflows present but the rule exists → gap (rule should be removed until a workflow lands)
   - Workflows present but the rule is absent → gap (rule should be added)
7. **Gaps** — bullet list of concrete deltas vs. baseline. Each gap should name the exact thing that's wrong (e.g. "ruleset bypass mode is `always` — should be `pull_request`", "`pull_request` rule's `allowed_merge_methods` is `[\"merge\",\"squash\",\"rebase\"]` — should be `[\"squash\"]`")

Facts and gaps only. No generic best-practice advice — the reader is the owner.
