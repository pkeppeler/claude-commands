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

## Step 0b — Token capability detection

Several endpoints below are admin-gated and 404 to non-admin tokens *without* indicating "you can't see this" — the 404 is indistinguishable from "the thing isn't set." Capture token capability **before** interpreting any 404, so unverifiable items don't get reported as gaps.

- **Repo admin**: read `permissions.admin` from the step 1 metadata response (true/false).
- **`admin:org` scope**: parse the `gh auth status` token-scopes line OR probe `gh api orgs/<owner.login>/rulesets` — a 404 whose body mentions `"needs the \"admin:org\" scope"` means the scope is missing.

Carry these two bits forward and use them to classify each admin-gated 404 as either "genuinely disabled/not set" or "unverifiable with this token." The following endpoints **require repo admin** — their 404s are ambiguous without it:

- `GET /repos/.../vulnerability-alerts`
- `GET /repos/.../private-vulnerability-reporting`
- `GET /repos/.../branches/.../protection`
- The `security_and_analysis` block on `GET /repos/...` is omitted entirely for non-admins (do not treat absence as "disabled")

The repo-scoped rulesets endpoint (`GET /repos/.../rulesets`) is readable by any `repo`-scoped token, so its results are trustworthy. The org-scoped endpoint (`GET /orgs/.../rulesets`) needs `admin:org`.

## What to inventory

### 1. Repo metadata — `gh api repos/<o>/<n>`
- `visibility`, `default_branch`, `owner.type`, `owner.login`
- Merge settings: `allow_squash_merge`, `allow_merge_commit`, `allow_rebase_merge`, `delete_branch_on_merge`, `allow_auto_merge`, `allow_update_branch`, `squash_merge_commit_title`, `squash_merge_commit_message`
- `security_and_analysis.{secret_scanning, secret_scanning_push_protection, dependabot_security_updates}.status` (may be absent on private/free)

`owner.plan.name` at this endpoint is often `null` for org-owned repos. Fetch the plan separately in step 1b.

### 1b. Owner plan — fetch separately

- `owner.type == "Organization"` → `gh api orgs/<owner.login> --jq .plan.name`
- `owner.type == "User"` → `gh api users/<owner.login> --jq .plan.name` (only populated when authenticated *as* that user)
- 404 / null → fall back to "unknown"; treat rulesets-list 200 vs 403 as a secondary probe (200 implies at least Team / Pro)

Map plan → feature availability:

| Plan | Rulesets on private | Vuln alerts / Dependabot updates / PVR | Secret scanning / push protection / code scanning |
|---|---|---|---|
| `free` | ❌ | ✅ free for all | ❌ private requires GHAS |
| `pro` (user) / `team` (org) | ✅ | ✅ | ❌ requires GHAS add-on |
| `enterprise` (or legacy `business_plus`) | ✅ | ✅ | ⚠️ only if GHAS add-on purchased — probe `security_and_analysis` |
| public visibility (any plan) | ✅ | ✅ | ✅ free |

Use this to classify each baseline item as `❌` (available but missing) vs `⏭️` (genuinely gated).

### 2. Vulnerability alerts — `gh api repos/<o>/<n>/vulnerability-alerts`
- 204 → enabled
- 404 + repo admin → disabled
- 404 + non-admin → **unverifiable** (admin-only endpoint)

### 3. Private Vulnerability Reporting — `gh api repos/<o>/<n>/private-vulnerability-reporting`
- Returns `{"enabled": true/false}` → use directly
- 404 + repo admin → unavailable on this plan
- 404 + non-admin → **unverifiable** (admin-only endpoint)

### 4. Rulesets

**Repo-scoped** — `gh api repos/<o>/<n>/rulesets` (any `repo` token):
- For each ruleset: name, target, enforcement, rules summary, bypass actors
- 403 → "not available on current plan" (private + free)
- Identify all rulesets whose `conditions.ref_name.include` contains `~DEFAULT_BRANCH` or the literal default branch name. Fetch full rule details with `gh api repos/<o>/<n>/rulesets/<id>` for each so individual rule parameters and bypass-actor modes are visible. More than one such ruleset is itself a gap (overlapping enforcement obscures effective policy).

**Org-scoped** — `gh api orgs/<owner.login>/rulesets` (only if `owner.type == "Organization"`):
- Needs `admin:org` scope. If the call 404s with a "needs the \"admin:org\" scope" body, mark org-scoped rulesets as **unverifiable** — do **not** assert "no enforcement" based solely on the empty repo-scoped list, since an org-scoped ruleset can target this repo invisibly.
- If accessible: list org rulesets whose `conditions` target this repo (by name, custom property, or `~ALL`) and whose `ref_name` conditions cover the default branch. Apply the same per-ruleset detail fetch as the repo-scoped case.

### 5. Legacy branch protection — `gh api repos/<o>/<n>/branches/<default_branch>/protection`
- 200 → list active protections
- 404 + repo admin → "not set"
- 404 + non-admin → **unverifiable** (admin-only endpoint)
- 403 → "not available"

### 6. Convention files — `gh api repos/<o>/<n>/contents/<path>`
Check presence (not content) of: `.github/CODEOWNERS`, `.github/dependabot.yml`, `SECURITY.md`, `.github/workflows/`, `.github/workflows/zizmor.yml`

### 7. Workflow job names — read `.github/workflows/*.yml` via `gh api`
For each workflow, list job keys and `name:` values. These are the candidate contexts for required status checks in a ruleset.

## First-class baseline

Compare findings against this baseline (mirrors what `/bootstrap-github-repo` applies):

**Repo settings**
- `allow_squash_merge: true`, `allow_merge_commit: false`, `allow_rebase_merge: false`
- `delete_branch_on_merge: true`
- `allow_auto_merge: true`, `allow_update_branch: true`
- `squash_merge_commit_title: PR_TITLE`, `squash_merge_commit_message: BLANK`

**Security** (classify against the plan map in step 1b — `❌` only when available on this plan)
- Vulnerability alerts: enabled — free for all plans
- Dependabot security updates: enabled — free for all plans
- Private Vulnerability Reporting: enabled — free for all plans
- Secret scanning: enabled — free on public; private requires GHAS (Enterprise + add-on, or Team + add-on)
- Push protection: enabled — same gating as secret scanning

**Default-branch ruleset** (named `default-branch-protection`)
- Enforcement: `active`
- `pull_request` rule: `required_approving_review_count: 1`, `dismiss_stale_reviews_on_push: true`, `require_code_owner_review: true`, `require_last_push_approval: false`, `required_review_thread_resolution: true`, `allowed_merge_methods: ["squash"]` (mirrors the repo-level squash-only policy as audit-visible defense in depth)
- `required_status_checks` rule — **conditional on workflow presence**:
  - Workflows exist → rule must be present with `strict_required_status_checks_policy: true` and contexts that exactly match the discovered job names (no missing, no stale)
  - No workflows → rule must be **absent** (a placeholder always-passing check is cargo cult)
- `non_fast_forward`, `deletion`, `required_linear_history` rules present
- Bypass actor: `RepositoryRole` admin (id 5) with **`bypass_mode: "pull_request"`** (admin can self-merge a PR without review, but **must still open a PR** — `"always"` is a gap because it permits direct pushes to main)

**Files**
- `.github/CODEOWNERS` exists with at least one `*` rule pointing at a user or team. (The bootstrap auto-creates `* @<owner.login>` only for repos owned by the authenticated user; org repos and cross-account admins are expected to populate this manually with the right team handle, so the audit doesn't pin a specific owner string.)
- `.github/dependabot.yml` exists (at minimum: `github-actions` ecosystem with a `cooldown` block — defense against supply-chain attacks; zizmor's `dependabot-cooldown` audit fails CI without it). Flag any ecosystem missing the cooldown block.
- `SECURITY.md` exists for public repos
- `.github/workflows/zizmor.yml` exists (GitHub Actions security audit — flags unpinned actions, excessive permissions, etc.)

**Legacy branch protection on default branch**
- Should be absent — the new ruleset replaces it. Presence (especially a no-op one) is a gap.

## Output

A single markdown report under ~500 words:

1. **Header** — repo, visibility, owner plan (from step 1b), default branch, and a one-line capability summary derived from the plan map (e.g. "private Team → rulesets available; GHAS-gated features require add-on", "private Free → no rulesets, no private GHAS", "public Pro → all features available").
   Add a **token capability line** immediately below: repo admin (yes/no) and `admin:org` scope (yes/no). When either is "no," follow with a one-line caveat listing what is therefore unverifiable (security toggles, legacy branch protection, org-scoped rulesets, as applicable).
2. **Settings table** — current value vs. baseline, with ✅/❌/⏭️ (skipped/unavailable) for each baseline setting above. Use **❓ (unverifiable)** in place of ❌ for any admin-gated item the token couldn't read.
3. **Rulesets** — list repo-scoped rulesets with summaries; for the default-branch ruleset, mark each baseline criterion ✅/❌, including the bypass mode. If org-scoped rulesets are unverifiable, say so explicitly — don't infer "no enforcement" from the empty repo-scoped list alone.
4. **Legacy branch protection** — present (and what it enforces), "not set" (only when token has repo admin), or "unverifiable (admin-only endpoint)".
5. **Files** — which baseline files are present / missing.
6. **Workflow jobs** — discovered job names. Compare against the ruleset's `required_status_checks` contexts and flag both directions:
   - Job exists in a workflow but not listed as a required context → gap (re-run bootstrap)
   - Context listed in the ruleset but no matching job in any workflow → gap (stale context, re-run bootstrap)
   - No workflows present but the rule exists → gap (rule should be removed until a workflow lands)
   - Workflows present but the rule is absent → gap (rule should be added)
   - If the default-branch ruleset is itself unverifiable (org-scoped + no `admin:org`), skip this cross-check and say so.
7. **Could not verify** — bullet list of items the token couldn't read, each naming the endpoint and the missing capability (e.g. "Legacy branch protection on `<default>` — admin-only endpoint, token lacks repo admin", "Org-scoped rulesets — `admin:org` scope not granted"). Include the remediation: re-run with an admin token / `gh auth refresh -s admin:org`.
8. **Gaps** — bullet list of concrete deltas vs. baseline. Each gap should name the exact thing that's wrong (e.g. "ruleset bypass mode is `always` — should be `pull_request`", "`pull_request` rule's `allowed_merge_methods` is `[\"merge\",\"squash\",\"rebase\"]` — should be `[\"squash\"]`", "multiple rulesets target the default branch (`default-branch-protection`, `legacy-protection`) — consolidate").
   **Never list an unverifiable item here.** If you couldn't read the endpoint, the right place is "Could not verify," not "Gaps" — the audit asserts only what it can prove.

Facts and gaps only. No generic best-practice advice — the reader is the owner.
