---
description: Apply first-class Python conventions (uv, ruff, pyright strict, pytest+cov, SHA-pinned CI, Dependabot). Works on fresh OR existing projects. All work happens on a branch.
---

Apply first-class Python conventions to the current directory. **Works on both fresh and existing projects.** Idempotent and merge-aware: missing files are created, existing files are patched in place where the merge is well-defined, and *destructive* changes (downgrading the Python pin, replacing a build backend, swapping linters, moving from flat to `src/` layout) are flagged for the user before any change.

**All mutations happen on a `bootstrap-python` branch** so the user can review the diff, run CI, and merge when ready.

**Package name** (the importable name): `$ARGUMENTS`.

If `$ARGUMENTS` is empty:
1. If `pyproject.toml` exists with `[project].name`, use that.
2. Otherwise snake_case the current directory name and confirm with the user.

NOTE: Python 3.14 is current as of 2026 — bump to whatever is the latest stable release at the time you run this. Update `requires-python` (as a series pin, e.g. `"==3.14.*"`), `[tool.pyright].pythonVersion`, and `[tool.ruff].target-version` together. **Do not** create a `.python-version` file — `requires-python` is the single source of truth; uv falls back to it for interpreter resolution.

---

## Step 0 — Preflight

Verify every tool the command will invoke **before** any mutation. Run the version checks in parallel. If any required check fails, stop and print the remediation — do **not** proceed to Step 1.

**Required tooling:**

- `uv --version` — install via `brew install uv` (macOS) or https://docs.astral.sh/uv/getting-started/installation/ if missing. The command is unusable without it.
- `git --version` — install via `brew install git` (macOS, if Xcode CLT isn't present), the platform package manager elsewhere, or https://git-scm.com/downloads. Bootstrap relies on git for the branch + commit flow.
- `git config user.email` **and** `git config user.name` — both must be set (locally or globally), otherwise Step 13's `git commit` will fail with a confusing "Please tell me who you are" error. If either is empty, stop and tell the user to set them: `git config --global user.email "you@example.com"` and `git config --global user.name "Your Name"`.

**Required repo state:**

- If the current directory isn't a git repo: run `git init` and proceed (commits will land on the default branch — branching requires a parent commit, so Step 2 is skipped on a fresh init).
- `git status --porcelain` — must be empty (clean working tree). Untracked files are fine; uncommitted modifications are not (they would get caught up in the bootstrap commit). If dirty, stop and ask the user to commit or stash first.

**Soft checks (note in the report; don't gate):**

- The target Python interpreter (e.g. 3.14) doesn't need to be pre-installed — `uv sync --dev` in Step 12 fetches it via `uv python install` on demand. If `uv python list --only-installed` doesn't show the target, just note "uv will install Python <version> during sync" so the user isn't surprised by network activity.

## Step 1 — Inventory (parallel reads)

Detect current state. Run reads in parallel via multiple tool uses in one message:

- `pyproject.toml` — parse `[project]`, `[build-system]`, `[tool.*]`, `[dependency-groups]`. Capture every key.
- `uv.lock`, `.envrc`, `.gitignore`
- `.python-version` — flag for removal if present (legacy; `requires-python` is the single source of truth now)
- Layout: `src/<pkg>/__init__.py`, `<pkg>/__init__.py` (flat), `src/<pkg>/py.typed`, `tests/__init__.py`
- Legacy toolchain markers: `requirements*.txt`, `setup.py`, `setup.cfg`, `Pipfile`, `Pipfile.lock`, `poetry.lock`, `.flake8`, `.isort.cfg`
- `.vscode/settings.json`, `.github/workflows/*.yml`, `.github/dependabot.yml`, `CLAUDE.md`

Print a one-line current-state summary, then a planned-changes list grouped as **create**, **patch**, **skip (already baseline)**, and **ask** (destructive deltas — see below). **Confirm with the user before continuing.**

### Destructive deltas — ask before changing

Surface each of these explicitly and wait for the user's go/no-go on each:

First **classify** the project's current toolchain (same buckets as `/audit-python`): `uv-native`, `PEP 621 (non-uv)`, `Poetry`, `Pipenv`, `setup.py (declarative)`, `setup.py (contains code)`, `setup.cfg`, `pip + requirements`, `mixed`. State the classification in the summary.

Then surface each of these explicitly and wait for go/no-go on each:

- **Toolchain migration** (when the classification is anything other than `uv-native` or `PEP 621`): the user is converting away from Poetry / Pipenv / pip / setup.py. Step 3 Phase A handles the mechanics — name the source toolchain in the prompt and tell the user what will be extracted, what will be deleted, and what the lockfile transition looks like. **If the user declines migration, stop the command** — most subsequent patches assume a uv-shaped `pyproject.toml`. Print: "Bootstrap is uv-native. Re-run after migrating manually, or accept the migration to proceed." For `setup.py (contains code)`, refuse outright (Step 3 Phase A explains why).
- **Build backend change** (when migrating, or on `PEP 621 (non-uv)` projects with `setuptools` / `poetry-core` / `flit` / `pdm-backend`): `[build-system]` will be rewritten to hatchling. May need package rebuilds for downstream consumers.
- **Linter swap**: `black` / `flake8` / `isort` configs → ruff. Listed separately because it can apply even on a uv-native project. Step 3 Phase A's linter sub-section spells out what gets removed and how custom rules are surfaced.
- **Layout move**: flat `<pkg>/` → `src/<pkg>/`. Don't auto-relocate code; recommend it and skip the `src/`-only files if declined.
- **Python pin raise**: existing `requires-python` is *lower* than the baseline target — confirm before raising. (If higher, leave it alone.)
- **Drop `.python-version`**: existing `.python-version` file — confirm before deleting. The convention is to keep `requires-python` in `pyproject.toml` as the single source of truth; uv reads it for interpreter resolution. If the user wants exact-patch pinning (e.g. `3.14.3`, which `requires-python` can't express precisely), let them keep `.python-version` and note it in the report.
- **Existing `[tool.pyright].typeCheckingMode != "strict"`**: turning on strict on an existing codebase will surface many errors. Confirm before flipping.
- **Existing CI workflow** that doesn't match baseline shape: don't replace — surface the deltas in the final report and let the user decide post-bootstrap.

## Step 2 — Branch

If commits exist on `HEAD`:

```
git switch -c bootstrap-python
```

If `bootstrap-python` already exists, append a numeric suffix (`bootstrap-python-2`, `-3`, ...) until the name is free.

If the repo has no commits yet (just `git init` or a brand-new repo), skip this step — work directly on the current default branch and the bootstrap will produce the initial commit.

All file mutations below run on this branch.

---

## Step 3 — `pyproject.toml`

If absent, create it with the full baseline below.

If present and the project is **already uv-native / PEP 621**, skip directly to **Phase B — Patch policy**.

If a **legacy toolchain** is detected (Poetry, Pipenv, pip + requirements, setup.py, setup.cfg) and the user confirmed migration in Step 1, run **Phase A — Migration** first to normalize `pyproject.toml` into uv shape, then run Phase B.

### Phase A — Legacy toolchain migration

**Cardinal rule**: read all legacy config first, build the new `pyproject.toml` in memory, write it atomically, *then* delete the legacy files. Never delete a legacy lockfile or config until the new file is written and parses.

#### Poetry → uv

Read from `[tool.poetry]`, write into `[project]`:

- `name`, `version`, `description`, `authors`, `license`, `readme`, `homepage`/`repository`/`documentation` (the URL fields fold into `[project.urls]`)
- `[tool.poetry.dependencies].python` → `[project].requires-python`. Translate the Poetry constraint, preferring the series-pin form (`==X.Y.*`) where the original constraint was a single major.minor:
  - `^3.11` → `>=3.11,<4` (preserve Poetry's "any 3.x ≥ 3.11" semantics)
  - `~3.11` → `==3.11.*` (Poetry's tilde means series-pin; use the wildcard form)
  - `>=3.11,<3.13` → preserve as-is (explicit range, can't simplify)
  - `"3.11"` (bare) → `==3.11.*`
- All other entries in `[tool.poetry.dependencies]` → `[project.dependencies]`. Translate Poetry version notation to PEP 440:
  - `"^X.Y.Z"` → `>=X.Y.Z,<(X+1).0.0` (or `<1.0.0` if the major is `0`, since Poetry treats `^0.x.y` as `>=0.x.y,<0.(x+1).0`)
  - `"~X.Y.Z"` → `>=X.Y.Z,<X.(Y+1).0`
  - `"X.Y.Z"` (bare) → `==X.Y.Z`
  - `"*"` → drop the version (PEP 440 leaves it unconstrained)
  - `{ version = "...", extras = ["a", "b"] }` → `pkg[a,b]>=...`
  - `{ version = "...", markers = "..." }` → `pkg>=...; <markers>`
  - `{ git = "...", rev/tag/branch = "..." }` → `pkg @ git+<url>@<rev>` in dependencies (PEP 440 direct reference)
  - `{ path = "..." }` → `pkg @ file://<absolute>` (or surface to user — local-path deps usually need attention)
- `[tool.poetry.group.<name>.dependencies]` → `[dependency-groups].<name>` (with the same constraint translation). The default `dev` group lands in `[dependency-groups].dev`.
- `[tool.poetry.scripts]` → `[project.scripts]` (same shape).
- Replace `[build-system]` with `requires = ["hatchling"]`, `build-backend = "hatchling.build"`.
- Remove every `[tool.poetry]` and `[tool.poetry.*]` table.
- Delete `poetry.lock`. (uv generates `uv.lock` in Step 12.)

If any constraint is too exotic to translate confidently (path deps, complex multi-source URLs, `optional = true` extras), leave it out, list it in the report, and ask the user to add it manually post-migration.

#### Pipenv → uv

`Pipfile` is TOML. Read:

- `[packages]` → `[project.dependencies]`. Pipenv uses pip-style specifiers already; preserve them. `"*"` → drop the version.
- `[dev-packages]` → `[dependency-groups].dev`.
- `[requires].python_version` → `[project].requires-python` as a series pin (e.g. `"3.11"` → `"==3.11.*"`).
- Pipfile carries no `name`/`version`/`description` — **ask the user** for these (don't invent them) before writing `[project]`.
- Add `[build-system]` hatchling.
- Delete `Pipfile` and `Pipfile.lock`.

#### pip + `requirements.txt` → uv

- **Ask the user** for `name`, `version`, `description` — none of these live in a requirements file.
- `requirements.txt` → `[project.dependencies]`. Skip lines starting with `-r`, `-e`, `--`, or comments (record them in the report; recursive includes and editable installs need user attention).
- `requirements-dev.txt` / `dev-requirements.txt` / `requirements/dev.txt` (whichever exists) → `[dependency-groups].dev`.
- Hashes (`--hash=...`) and pins from `pip-compile` output — preserve the package + version specifier; drop the hashes (uv.lock will produce its own).
- Add `[build-system]` hatchling.
- **Don't delete `requirements*.txt` files until Step 12's `uv lock` succeeds.** If lock fails, the user has the originals to fall back to. After success, delete and note it in the report.

#### `setup.py` → uv

First classify the file:

- **Declarative** — only contains `from setuptools import setup` (or similar) and a single `setup(...)` call with literal kwargs. Safe to migrate.
- **Contains code** — imports beyond setuptools, conditional logic, dynamic version (`get_version()`, reads from VCS), custom commands, entry points built at runtime. **Refuse to auto-migrate.** Print: "`setup.py` contains code beyond a literal `setup()` call. Bootstrap can't safely extract metadata. Convert manually to `[project]` first, then re-run." Then exit Phase A without touching anything.

For declarative `setup.py`, parse with `ast` (never `exec`) and extract:

- `name`, `version`, `description`, `long_description` (if a literal — if it's `open("README.md").read()`, point `[project].readme = "README.md"` instead), `author`, `author_email`, `url`, `license`
- `python_requires` → `[project].requires-python`
- `install_requires` → `[project.dependencies]`
- `extras_require` → `[project.optional-dependencies]`
- `entry_points = {"console_scripts": [...]}` → `[project.scripts]`
- `packages` / `package_dir` — used to confirm layout; report deltas vs. `src/` baseline

After extraction, delete `setup.py`. Replace `[build-system]` with hatchling.

#### `setup.cfg` → uv

Mostly the same fields as setup.py, in INI form:

- `[metadata]` → `[project]` (`name`, `version`, `description`, `author`, `author_email`, `url`, `license`, `long_description = file: README.md` → `readme = "README.md"`)
- `[options].install_requires` → `[project.dependencies]`
- `[options].python_requires` → `[project].requires-python`
- `[options.extras_require]` → `[project.optional-dependencies]`
- `[options.entry_points].console_scripts` → `[project.scripts]`

Delete `setup.cfg` after extraction. If `setup.py` is *also* present and is a stub (`from setuptools import setup; setup()`), delete it too.

#### Linter migration (independent of dependency toolchain)

Run regardless of source toolchain when these are detected:

- **`black` / `[tool.black]`** → ruff format covers it. Remove `[tool.black]` from `pyproject.toml`. If `[tool.black].line-length` differs from the baseline `100`, surface and ask before normalizing.
- **`flake8` / `.flake8` / `[tool.flake8]`** → ruff covers it, but custom `extend-select` / `extend-ignore` rules may not have ruff equivalents. Show the user the existing config and ask before deleting; offer to translate known-equivalent rules into `[tool.ruff.lint].select` / `.ignore`.
- **`isort` / `.isort.cfg` / `[tool.isort]`** → ruff's `I` rule covers it. Remove `[tool.isort]` and `.isort.cfg`. If `known-first-party` is set, fold it into `[tool.ruff.lint.isort].known-first-party`.

For each linter migration, list what was removed (with the original config block in the report) so the user can re-add anything custom that didn't translate.

#### After Phase A

`pyproject.toml` is now uv-shaped (PEP 621 `[project]`, hatchling `[build-system]`, `[dependency-groups]`). Phase B then patches the tool config (`[tool.pyright]`, `[tool.ruff]`, etc.) on top.

Don't run `uv lock` here — let Step 12 do it. If the migration produced bad constraints, the failure surfaces in Step 12 where it's expected and the report is structured to surface it cleanly.

### Phase B — Patch policy

Apply to the (now uv-shaped) `pyproject.toml`:

- **`[build-system]`** — if absent, add `requires = ["hatchling"]`, `build-backend = "hatchling.build"`. If present and uses hatchling, no-op. If different, this is the destructive delta from Step 1 — only change after user confirmation.
- **`[project].requires-python`** — set if absent, using the series-pin form (`"==<major>.<minor>.*"`). If present and *lower* than baseline, raise it (with user confirmation), writing the new value in series-pin form. If present and *higher*, leave it. If present and already in series-pin form, leave the format alone.
- **`[project].name`, `version`, `description`, `authors`, `dependencies`, `optional-dependencies`** — leave alone. Bootstrap doesn't own project metadata.
- **`[dependency-groups].dev`** — union with baseline `["pyright", "pytest", "pytest-cov", "ruff"]`. Don't touch existing pins or extra entries.
- **`[tool.pyright]`** — set `pythonVersion`, `typeCheckingMode = "strict"` (subject to user confirmation if currently looser). Leave any other keys alone.
- **`[tool.ruff]`** — set `line-length = 100` and `target-version` if absent. If present with different values, leave them and report the delta.
- **`[tool.ruff.lint].select`** — union with baseline rule families: `E`, `W`, `F`, `I`, `UP`, `B`, `SIM`, `C4`, `S`, `RUF`. Don't remove any rule family the user has added.
- **`[tool.ruff.lint.per-file-ignores]`** — add `"tests/**" = ["S101"]` if absent (additive — keep existing entries).
- **`[tool.ruff.lint.isort].known-first-party`** — add `<package>` to the list if absent.
- **`[tool.pytest.ini_options]`** — set `testpaths = ["tests"]` if absent. For `addopts`, ensure `--cov=<package>` and `--cov-report=term-missing` are present (append to existing string if needed; don't replace). Set `xfail_strict = true` and `filterwarnings = ["error"]` if absent — a passing `xfail` becomes a failure (no silent xpass), and warnings become test failures so deprecation/library warnings get addressed when they appear, not after they pile up.
- **`[tool.coverage.run]`** — set `branch = true`, `source = ["src/<package>"]` if absent.
- **`[tool.coverage.report].exclude_also`** — union with `["if TYPE_CHECKING:", "raise NotImplementedError"]`.

Baseline (used when creating from scratch — note: no `fail_under` threshold yet, add one (e.g. 70) once the project has enough real tests to make it meaningful):

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "<package>"
version = "0.1.0"
requires-python = "==3.14.*"

[dependency-groups]
dev = ["pyright", "pytest", "pytest-cov", "ruff"]

[tool.pyright]
pythonVersion = "3.14"
typeCheckingMode = "strict"

[tool.ruff]
line-length = 100
target-version = "py314"

[tool.ruff.lint]
select = ["E", "W", "F", "I", "UP", "B", "SIM", "C4", "S", "RUF"]

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]

[tool.ruff.lint.isort]
known-first-party = ["<package>"]

[tool.pytest.ini_options]
addopts = "--cov=<package> --cov-report=term-missing"
testpaths = ["tests"]
xfail_strict = true
filterwarnings = ["error"]

[tool.coverage.run]
branch = true
source = ["src/<package>"]

[tool.coverage.report]
exclude_also = ["if TYPE_CHECKING:", "raise NotImplementedError"]
```

## Step 4 — Version pin coherence

After Step 3, re-read the three pins (`requires-python`, `[tool.pyright].pythonVersion`, `[tool.ruff].target-version`) and confirm they agree on the same major.minor. If they disagree, normalize to the highest version found (after user confirmation if it raises any of them).

If a `.python-version` file is present, surface it in the report — recommend deleting it (after Step 1's destructive-delta confirmation) so `requires-python` is the single source of truth. The only reason to keep `.python-version` is if the user wants an exact-patch pin (e.g. `3.14.3`) that `requires-python` can't express precisely.

## Step 5 — Layout files

- **`src/<package>/__init__.py`** — create empty if missing. Don't touch if present.
- **`src/<package>/py.typed`** — create empty marker if missing. Tells Pyright/Pylance to enforce types for downstream consumers.
- **`tests/__init__.py`** — create empty if missing.
- **Flat `<package>/` exists, no `src/<package>/`** — surface as a destructive delta (Step 1). If user declines the move, skip the `src/`-specific files for this run.

## Step 6 — `.gitignore`

If absent, create with the full baseline below. If present, **append** any missing baseline entries (don't reorder or rewrite the existing file).

```
__pycache__/
*.py[oc]
build/
dist/
*.egg-info/
.venv
.coverage
htmlcov/
```

Project-specific secret paths: ask the user once if there are any to add; don't guess.

## Step 7 — `.envrc`

Create with `source .venv/bin/activate` only if missing. Don't modify existing `.envrc` files — direnv setups vary.

## Step 8 — `.vscode/settings.json`

If absent, create with the full baseline below.

If present, **deep-merge**: set the keys below if they're missing or set to a different value. Preserve every other key the user has configured.

```json
{
  "python.analysis.typeCheckingMode": "strict",
  "python.analysis.importFormat": "absolute",
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  }
}
```

## Step 9 — `.github/workflows/ci.yml`

If `.github/workflows/` is absent or contains no `ci.yml`, create the baseline below.

If a workflow already exists that looks like CI (jobs named `lint`/`typecheck`/`test`, or running ruff/pyright/pytest), **don't replace it**. Report the deltas vs. baseline and let the user decide. Common deltas worth flagging:

- Actions not pinned by 40-char SHA (any `@vN` or `@main`)
- Missing top-level `permissions:` block (defaults to write-access on `GITHUB_TOKEN`)
- Missing `with: persist-credentials: false` on `actions/checkout` steps
- Missing `uv lock --check` step
- Missing `enable-cache: true` on `astral-sh/setup-uv`
- Jobs run sequentially in one job instead of three parallel jobs

Baseline shape (illustrative — **look up the current SHA for each action's latest release before writing; do not invent SHAs**):

Three jobs (`lint`, `typecheck`, `test`) running in parallel on `pull_request` and `push` to `main`.

**Top-level workflow defaults:**

- `permissions: { contents: read }` — minimal token scope; lint/typecheck/test never push or comment.
- Every `actions/checkout` step uses `with: { persist-credentials: false }` so the workflow doesn't leave the auth token persisted in `.git/config` after checkout (defense-in-depth against artifact-credential leaks).

**Each job:**

- `actions/checkout` (SHA-pinned with `# vX.Y.Z` comment) + `persist-credentials: false`
- `astral-sh/setup-uv` (SHA-pinned) with `enable-cache: true`
- `uv lock --check`
- `uv sync --dev`
- Then the job-specific command:
  - `lint`: `uv run ruff check` then `uv run ruff format --check`
  - `typecheck`: `uv run pyright`
  - `test`: `uv run pytest`

> The GitHub Actions security audit (zizmor) lives in a separate
> `.github/workflows/zizmor.yml` workflow created by `/bootstrap-github-repo` —
> it's language-agnostic and shouldn't be mixed into the language-specific
> CI here. If a project ran `/bootstrap-github-repo` first, zizmor will be
> auditing `ci.yml` for you on every PR.

## Step 10 — `.github/dependabot.yml`

If absent, create with the baseline below.

If present, **patch**: ensure both `pip` and `github-actions` ecosystem entries exist with `interval: "weekly"`. If `pip` exists without grouping, add the minor/patch group. Don't remove or modify any other ecosystem entries the user has configured.

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "pip"   # reads pyproject.toml; works for uv projects
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      python-minor-patch:
        update-types: ["minor", "patch"]
```

## Step 11 — `CLAUDE.md`

If absent, create with the Python Toolchain block below as the only content.

If present, check for a Python Toolchain section. If missing, **append** the block. If present, leave it alone — don't overwrite the user's customizations.

```markdown
## Python Toolchain

- Python 3.14 (pinned via `requires-python = "==3.14.*"` in `pyproject.toml`; uv installs the latest 3.14.x automatically)
- `uv` for env + dependencies. Lockfile (`uv.lock`) is committed.
- `src/` layout with `hatchling` build backend.
- Runtime deps in `[project.dependencies]`, dev tools in `[dependency-groups].dev`.

Common commands:
- `uv sync --dev` — install everything
- `uv run pyright` — type check
- `uv run ruff check` / `uv run ruff format` — lint and format
- `uv run pytest` — tests with coverage
- `uv lock --check` — verify lockfile is current

## Type Safety

- **Pyright strict mode** (`typeCheckingMode = "strict"`). Zero errors required.
- **Pydantic for all structured data** — tool params/returns, config, external API responses, and internal helpers alike. No `dataclass`, no `dict[str, Any]` floating around as a stand-in for a real type. Consistency beats the marginal ergonomic win of dataclasses, and Pydantic v2's Rust core makes the runtime cost negligible. `TypedDict` is allowed only when a dict shape is genuinely required at a boundary (e.g. `**kwargs` forwarding, third-party APIs that demand a dict) — otherwise use a model and `model_dump()`.
- **`py.typed` marker** ships with the package so consumers honor types.
- **`__all__` on modules with a public API.** Declares what's exported and stops `json`/`httplib2`/etc. from showing up as auto-import candidates. Internal-only modules and tests don't need it.
- `# type: ignore[<specific-rule>]` only at third-party boundaries with missing stubs. Never blanket-ignore.

## Linting & Formatting

- **Ruff** handles linting, formatting, and import sorting (`[tool.ruff]` in `pyproject.toml`).
- Active rule families: pycodestyle, pyflakes, isort, pyupgrade, bugbear, simplify, comprehensions, bandit (security), and ruff's own rules. Tests get `S101` (asserts) waived.
- VS Code is configured to run `ruff format` on save and apply auto-fixes + organize imports.
- CI fails on `ruff check` errors or unformatted files.
- `# noqa: <specific-rule>` only at narrow, justified spots — same discipline as `# type: ignore[<rule>]`. No blanket ignores.

## Imports

- **Import from the defining module, not from re-exports.** If `pkg.foo` defines `Bar`, write `from pkg.foo import Bar` — not `from pkg import Bar`, even if `pkg/__init__.py` re-exports it.
  - Makes inter-module dependencies explicit and grep-able for refactors.
  - Avoids circular-import traps that re-exports can hide.
  - Pairs with `python.analysis.importFormat = "absolute"` in `.vscode/settings.json`.
- This applies to *first-party* code. Third-party packages with curated public APIs (e.g. `from pydantic import BaseModel`) follow their documented import path.

## Testing

- `uv run pytest` runs the suite with coverage enabled (`--cov` configured in `pyproject.toml`).
- Branch coverage is on. Lines like `if TYPE_CHECKING:` and `raise NotImplementedError` are excluded from the report.
- No hard `fail_under` threshold yet — add one once the project has enough tests for it to be a meaningful gate.
- New behavior should land with tests covering it.

## CI

GitHub Actions runs three jobs in parallel on every PR and push to main: `lint` (ruff), `typecheck` (pyright), and `test` (pytest with coverage). All must pass to merge. The uv cache is enabled so reruns are fast.

## Supply Chain Security

- **Pin GitHub Actions by SHA, not tag.** Tags are mutable; SHAs are not. Format: `uses: actions/checkout@<40-char-sha> # v4.3.1`
- **Dependabot** opens weekly PRs for both GitHub Actions and Python deps. Minor/patch Python updates are grouped into a single PR; majors come individually.
- Never use `@main`, `@master`, or `@latest`.

## Editor

`.vscode/settings.json` is committed:
- Pylance strict type checking (matches CI)
- `python.analysis.importFormat = "absolute"` — imports resolve to the source module, not re-exports
- Ruff as default formatter, format-on-save, fix-all and organize-imports on save

## Code Style

- Comment only when the *why* is non-obvious (a hidden constraint, a workaround, surprising behavior). Default to no comments — well-named identifiers do the work.
- Ruff handles formatting and import sorting — don't hand-format.
```

---

## Step 12 — Verify on the branch

Run all four, **on the bootstrap branch**:

1. `uv sync --dev` — materialize the venv and lockfile (this is also where a botched migration first surfaces — bad version constraints from a Poetry/Pipenv translation will fail to resolve here)
2. `uv run pyright`
3. `uv run ruff check` then `uv run ruff format --check`
4. `uv run pytest`

For a fresh project, all four should exit clean. **For migrations, don't treat non-zero exits as bootstrap failure** — surface the counts (e.g. "pyright: 47 errors", "ruff format: 12 files would be reformatted") and report them. Existing code surfacing strict-mode violations is expected; the user fixes those incrementally after the bootstrap lands.

**If `uv sync --dev` fails** on a migration: do **not** delete the legacy lockfiles or requirements files (Step 3 Phase A's "delete after lock succeeds" rule). The user needs the originals to debug. Surface the resolver error verbatim and stop before Step 13 — the migration produced bad constraints and needs human review. Common causes: a Poetry caret on `0.x.y` mistranslated, a path/git dep that didn't get a direct reference, or a transitive constraint conflict that Poetry was hiding.

**Only after `uv sync --dev` succeeds**, perform the deferred legacy-file deletions Phase A scheduled (`requirements*.txt`, etc.) and add them to the same commit.

## Step 13 — Commit on the branch

Stage the bootstrap changes and commit. Use a single commit (one reviewable diff):

```
git add -A
git commit -m "Bootstrap Python conventions (uv, ruff, pyright strict, pytest+cov, CI)"
```

Don't push. Don't open a PR. The user reviews the branch and decides.

## Step 14 — Final report

Tell the user:

1. The branch name (`bootstrap-python` or the suffixed variant)
2. **Source toolchain** detected and (if applicable) **migration summary** — what was extracted, what was deleted, and any constraints that didn't translate cleanly (path deps, exotic Poetry sources, requirements-file `-r`/`-e` lines). For each untranslated item, name the package and what the user needs to add manually.
3. **Created** files (list)
4. **Patched** files (list with one-line summaries of what changed)
5. **Skipped** items (already at baseline)
6. **Asked / declined** items (destructive deltas the user opted out of)
7. **Verification results** — exit status + summary line for each of the four tools
8. How to review and merge:
   - `git diff main..bootstrap-python` (or whatever the base branch is) to see the full diff
   - `git switch main && git merge --ff-only bootstrap-python` after CI passes (or open a PR)
   - Re-run `/audit-python` to confirm the baseline is satisfied

End with any per-project rules to add to `CLAUDE.md` that the bootstrap couldn't infer (project-specific secret paths, domain conventions, etc.).
