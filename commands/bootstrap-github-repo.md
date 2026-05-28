---
description: Apply first-class GitHub repo settings, security features, ruleset, CODEOWNERS, SECURITY.md, dependabot.yml, and a zizmor workflow security audit (idempotent, plan-aware)
---

Apply a first-class GitHub configuration to a repository. **Idempotent — safe to re-run.** Re-run after adding, renaming, or removing workflow jobs so the default-branch ruleset's required status checks stay in sync.

**Target repo**: `$ARGUMENTS` in `owner/name` form. If empty, derive from the current directory using `gh repo view --json nameWithOwner --jq .nameWithOwner`. **Confirm the target with the user before any mutation.**

**Authenticated user login** (used to gate CODEOWNERS auto-creation): `gh api user --jq .login`.

Use `gh api` throughout. Reads in parallel; writes sequential so failures are easy to attribute.

---

## Step 0 — Preflight

Verify `gh` is installed and authenticated **before** any other work.

- `gh --version` — must succeed. If not, stop and tell the user:
  - macOS: `brew install gh`
  - Linux/Windows: see https://github.com/cli/cli#installation
- `gh auth status` — must show authenticated, and the authenticated user must have `admin` permission on the target repo (the mutations in steps 2–6 require it). If not authenticated, tell the user to run `gh auth login` or set `GH_TOKEN` / `GITHUB_TOKEN`. If authenticated but not an admin on the target repo, stop and explain — bootstrapping requires admin to change settings, security toggles, and rulesets.

If any check fails, **do not proceed**. Print the remediation and exit before any mutating call.

---

## Step 1 — Inventory (read, parallel)

Fetch:
- `gh api repos/<o>/<n>` — settings, visibility, `owner.type`, `owner.login`
- `gh api repos/<o>/<n>/vulnerability-alerts`
- `gh api repos/<o>/<n>/private-vulnerability-reporting`
- `gh api repos/<o>/<n>/rulesets` — also serves as a secondary plan probe (see below)
- `gh api repos/<o>/<n>/branches/<default>/protection`
- `gh api repos/<o>/<n>/contents/.github/workflows` — 404 if absent. If present, iterate and fetch each file via `gh api repos/<o>/<n>/contents/.github/workflows/<name>`, base64-decode the `content` field, and parse for job keys and `name:` values.

Then fetch the owner's plan (the `repos/<o>/<n>` response usually returns `owner.plan: null`):
- `owner.type == "Organization"` → `gh api orgs/<owner.login> --jq .plan.name`
- `owner.type == "User"` → `gh api users/<owner.login> --jq .plan.name` (only populated when authenticated *as* that user)
- null / 404 → fall back to "unknown"

Determine feature availability from visibility + plan:

| Plan | Rulesets on private | Vuln alerts / Dependabot updates / PVR | Secret scanning / push protection |
|---|---|---|---|
| `free` | ❌ | ✅ free for all | ❌ requires GHAS |
| `pro` (user) / `team` (org) | ✅ | ✅ | ❌ requires GHAS add-on |
| `enterprise` (or legacy `business_plus`) | ✅ | ✅ | ⚠️ only if GHAS add-on purchased — probe empirically in step 3 |
| public visibility (any plan) | ✅ | ✅ | ✅ free |

When plan is "unknown", use rulesets-list 200 vs 403 as the canary (200 ≥ Team/Pro, 403 = Free on private). Probe GHAS features empirically (422 → skip).

Print a one-line plan summary like *"private Team — rulesets available; GHAS-gated features will be skipped unless add-on is detected"* so the user knows what's coming before any mutation.

---

## Step 2 — Repo settings (always available)

`gh api -X PATCH repos/<o>/<n>` with:

```json
{
  "allow_squash_merge": true,
  "allow_merge_commit": false,
  "allow_rebase_merge": false,
  "delete_branch_on_merge": true,
  "allow_auto_merge": true,
  "allow_update_branch": true,
  "squash_merge_commit_title": "PR_TITLE",
  "squash_merge_commit_message": "BLANK"
}
```

---

## Step 3 — Security toggles

- **Vulnerability alerts**: `gh api -X PUT repos/<o>/<n>/vulnerability-alerts` (204 OK on success or already-on)
- **Dependabot security updates**: `gh api -X PUT repos/<o>/<n>/automated-security-fixes`
- **Private Vulnerability Reporting**: `gh api -X PUT repos/<o>/<n>/private-vulnerability-reporting`
- **Secret scanning + push protection**: `gh api -X PATCH repos/<o>/<n>` with:

  ```json
  {
    "security_and_analysis": {
      "secret_scanning": { "status": "enabled" },
      "secret_scanning_push_protection": { "status": "enabled" }
    }
  }
  ```

Catch errors from features unavailable on the current plan (typically 422 or 403) and record them as "skipped: requires X". Do not abort the command.

---

## Step 4 — Baseline files (create only if missing)

Check existence with `gh api repos/<o>/<n>/contents/<path>`. To create, use `gh api -X PUT repos/<o>/<n>/contents/<path>` with a base64-encoded `content` field and a `message`. **Never overwrite existing content.**

**`.github/CODEOWNERS`** — only auto-create when the repo is owned by the authenticated user. The bootstrap can't infer the right value in any other case, and writing the wrong owner is worse than writing nothing (silently makes the bootstrapper a CODEOWNER of someone else's repo, or — for orgs — produces a `* @user` entry that GitHub may silently ignore if the user lacks write access, leaving `require_code_owner_review` as a no-op):

- `owner.type == "User"` **and** `owner.login == <authenticated user>` (from step 1's metadata) → write:
  ```
  * @<owner.login>
  ```
- `owner.type == "User"` **and** mismatch → skip. Print: "Repo owned by `<owner.login>`, but you're authenticated as `<you>`. Skipping CODEOWNERS — create it manually with the right user."
- `owner.type == "Organization"` → skip. Print: "Org repo. CODEOWNERS should reference a team handle like `@<org>/<team>`, not a person. Skipping; create it manually."

**`SECURITY.md`** (public repos only — skip for private):
```markdown
# Security Policy

To report a vulnerability, please use GitHub's **"Report a vulnerability"** button on the [Security tab](../../security) of this repository. This opens a private security advisory — please do not open a public issue for security bugs.

You can expect an initial response within a few days.
```

**`.github/dependabot.yml`** (only if missing):
```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
```

The `cooldown` block delays Dependabot from opening PRs for new versions until 7 days post-release — defense against supply-chain attacks (typosquatting, malicious-then-yanked packages). zizmor's `dependabot-cooldown` audit flags ecosystems without this; failing to include it will fail the zizmor CI check.

Tell the user explicitly: language-specific ecosystems (`uv` for Python, `npm`, etc.) should be added to this file separately, or via `/bootstrap-python` and friends. Each added ecosystem should also include a `cooldown` block.

**`.github/workflows/zizmor.yml`** (only if missing — GitHub Actions security audit, language-agnostic):

This workflow runs [zizmor](https://docs.zizmor.sh) on every push to `main` and every PR. zizmor enforces the SHA-pinning convention and catches other workflow-security issues (excessive permissions, credential persistence, untrusted-input usage, etc.). Owning this workflow at the github-bootstrap level (rather than mixing it into language-specific CI) means non-Python repos still get the security audit.

Latest zizmor is installed each run — **do not pin** the version (a security tool with stale audits defeats the purpose).

```yaml
name: zizmor

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  zizmor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha> # vX.Y.Z
        with:
          persist-credentials: false
      - uses: astral-sh/setup-uv@<sha> # vX.Y.Z
        with:
          enable-cache: true
      - run: uv tool install zizmor
      - run: zizmor .
```

**Look up the current SHAs for `actions/checkout` and `astral-sh/setup-uv` before writing — do not invent SHAs.** Use the same SHA-pinning format (`@<40-char-sha> # vX.Y.Z`) the bootstrap enforces elsewhere.

If `zizmor .` flags pre-existing findings in OTHER workflows on first run, those need fixing in the workflows themselves (the bootstrap doesn't auto-fix them — that's the user's call). The most common findings on legacy workflows: missing `permissions:` blocks (severity: medium) and `persist-credentials: false` on checkout (severity: medium). Surface the finding counts in the final report so the user knows what to expect.

Once `zizmor.yml` exists, step 1's workflow discovery will pick up its `zizmor` job key, and step 5's ruleset will include it as a required status check automatically.

**If the contents PUT fails with 409/422**: a default-branch ruleset is already enforcing against direct commits (admin bypass mode `pull_request` blocks the API write). Record as "skipped: ruleset blocks direct commit — create `<path>` via PR" and continue. This typically only happens on a re-run where the ruleset already exists but a baseline file is missing; on the normal first-run path, Step 4 finishes before Step 5 creates the ruleset.

---

## Step 5 — Default-branch ruleset (skip if plan disallows)

Use the workflow job names already discovered in step 1. A job's reported context is its `name:` if set, otherwise its YAML key under `jobs:`. Include all of them.

**Conditional rule**: include the `required_status_checks` rule **only if** at least one workflow job was discovered. If `.github/workflows/` is absent or has no jobs, omit the rule entirely — a placeholder no-op check is cargo cult, and an empty `required_status_checks` array provides no signal. The rule will be added on a subsequent re-run once a workflow exists.

**Check for overlapping rulesets** before mutating: from step 1's ruleset list, fetch full details (`gh api repos/<o>/<n>/rulesets/<id>`) for any whose `conditions.ref_name.include` contains `~DEFAULT_BRANCH` or the literal default branch name. If any exist beyond `default-branch-protection`, surface them: "Found N additional ruleset(s) targeting the default branch: [name1, name2]. They will continue to enforce alongside this ruleset; consider consolidating manually." Then continue.

Find any existing ruleset named `default-branch-protection`:
- Exists → `gh api -X PUT repos/<o>/<n>/rulesets/<id>` (replace)
- Missing → `gh api -X POST repos/<o>/<n>/rulesets` (create)

Ruleset payload (omit the `required_status_checks` rule object when no workflow jobs were discovered; expand `required_status_checks` to one `{ "context": "..." }` entry per discovered job — placeholders below are illustrative):

```json
{
  "name": "default-branch-protection",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] }
  },
  "rules": [
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": false,
        "required_review_thread_resolution": true,
        "allowed_merge_methods": ["squash"]
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "<job-name-1>" },
          { "context": "<job-name-2>" }
        ]
      }
    },
    { "type": "non_fast_forward" },
    { "type": "deletion" },
    { "type": "required_linear_history" }
  ],
  "bypass_actors": [
    { "actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "pull_request" }
  ]
}
```

`allowed_merge_methods: ["squash"]` mirrors the repo-level squash-only policy from step 2. Belt-and-suspenders: repo settings can be flipped with one admin click, but the ruleset is the audit-visible source of truth for branch policy.

`actor_id: 5` is the admin repository role. Bypass mode `pull_request` means admins can self-merge a PR without requiring an approving review, **but still must open a PR** — direct pushes to the default branch are rejected for everyone, including admins. This is the right shape for "always use PRs; the owner doesn't need a reviewer."

If the API returns 403 with an "Upgrade" message, this is private + Free — record as "skipped: rulesets on private repos require GitHub Pro (user) or Team (org)" and continue.

---

## Step 6 — Legacy branch protection cleanup

If step 1 found legacy branch protection on the default branch:

- **No-op** (no required checks, no required reviews, no force-push block) → `gh api -X DELETE repos/<o>/<n>/branches/<default>/protection`. The new ruleset replaces it cleanly.
- **Has real rules** → do NOT delete. Report what it enforces and recommend the user remove it manually once the ruleset is proven out.

---

## Step 7 — Verification read-back

Re-fetch the same endpoints from step 1 and produce a final markdown report — same shape and word cap as `/audit-github-repo` (under ~500 words): header (repo / visibility / plan), settings table, security toggles, ruleset summary, legacy branch protection, files, workflow jobs vs. required contexts, and a result line per category using:

- ✅ — now configured per baseline
- ⏭️ — skipped due to plan limits (with the reason)
- ❌ — failed for another reason (include the error message)

If overlapping default-branch rulesets were detected in step 5, surface that here as well.

End with: "Re-run `/audit-github-repo <owner/name>` anytime to verify."

---

## Important

- Confirm the target repo with the user before the first mutation.
- Catch and surface plan-related errors as "skipped: requires X" — don't fail the whole command.
- Never overwrite an existing `dependabot.yml`, `SECURITY.md`, `CODEOWNERS`, or `.github/workflows/zizmor.yml`. Only create if missing.
- The command should be safe to run multiple times. Re-runs converge state to the baseline.
