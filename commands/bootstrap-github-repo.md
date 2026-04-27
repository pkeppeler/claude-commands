---
description: Apply first-class GitHub repo settings, security features, ruleset, CODEOWNERS, SECURITY.md, and dependabot.yml (idempotent, plan-aware)
---

Apply a first-class GitHub configuration to a repository. **Idempotent — safe to re-run.** Re-run after adding, renaming, or removing workflow jobs so the default-branch ruleset's required status checks stay in sync.

**Target repo**: `$ARGUMENTS` in `owner/name` form. If empty, derive from the current directory using `gh repo view --json nameWithOwner --jq .nameWithOwner`. **Confirm the target with the user before any mutation.**

**Owner login** for CODEOWNERS: `gh api user --jq .login`.

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
- `gh api repos/<o>/<n>` — settings, visibility, plan
- `gh api repos/<o>/<n>/vulnerability-alerts`
- `gh api repos/<o>/<n>/private-vulnerability-reporting`
- `gh api repos/<o>/<n>/rulesets`
- `gh api repos/<o>/<n>/branches/<default>/protection`
- Workflow files in `.github/workflows/*.yml` (parse for job names)

Determine plan constraints:
- **Public** → all features available
- **Private + free** (`owner.plan.name == "free"`) → no rulesets, no branch protection, no Code/Secret/Push-Protection scanning
- **Private + paid** → rulesets available; scanning depends on GHAS

Print a one-line plan summary so the user knows what will be skipped.

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

**`.github/CODEOWNERS`**:
```
* @<owner-login>
```

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
```

Tell the user explicitly: language-specific ecosystems (`pip`, `npm`, etc.) should be added to this file separately, or via `/bootstrap-python` and friends.

**If the contents PUT fails with 409/422**: a default-branch ruleset is already enforcing against direct commits (admin bypass mode `pull_request` blocks the API write). Record as "skipped: ruleset blocks direct commit — create `<path>` via PR" and continue. This typically only happens on a re-run where the ruleset already exists but a baseline file is missing; on the normal first-run path, Step 4 finishes before Step 5 creates the ruleset.

---

## Step 5 — Default-branch ruleset (skip if plan disallows)

Discover required status check contexts: read each `.github/workflows/*.yml` and collect job contexts. A job's reported context is its `name:` if set, otherwise its YAML key under `jobs:`. Include all of them.

**Conditional rule**: include the `required_status_checks` rule **only if** at least one workflow job was discovered. If `.github/workflows/` is absent or has no jobs, omit the rule entirely — a placeholder no-op check is cargo cult, and an empty `required_status_checks` array provides no signal. The rule will be added on a subsequent re-run once a workflow exists.

Find any existing ruleset named `default-branch-protection`:
- Exists → `gh api -X PUT repos/<o>/<n>/rulesets/<id>` (replace)
- Missing → `gh api -X POST repos/<o>/<n>/rulesets` (create)

Ruleset payload (omit the `required_status_checks` rule object when no workflow jobs were discovered):

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

If the API returns 403 with an "Upgrade" message, this is private + free — record as "skipped: rulesets require GitHub Pro" and continue.

---

## Step 6 — Legacy branch protection cleanup

If step 1 found legacy branch protection on the default branch:

- **No-op** (no required checks, no required reviews, no force-push block) → `gh api -X DELETE repos/<o>/<n>/branches/<default>/protection`. The new ruleset replaces it cleanly.
- **Has real rules** → do NOT delete. Report what it enforces and recommend the user remove it manually once the ruleset is proven out.

---

## Step 7 — Verification read-back

Re-fetch the same endpoints from step 1 and produce a final markdown report:

- ✅ — now configured per baseline
- ⏭️ — skipped due to plan limits (with the reason)
- ❌ — failed for another reason (include the error message)

End with: "Re-run `/audit-github-repo <owner/name>` anytime to verify."

---

## Important

- Confirm the target repo with the user before the first mutation.
- Catch and surface plan-related errors as "skipped: requires X" — don't fail the whole command.
- Never overwrite an existing `dependabot.yml`, `SECURITY.md`, or `CODEOWNERS`. Only create if missing.
- The command should be safe to run multiple times. Re-runs converge state to the baseline.
