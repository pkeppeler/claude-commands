---
description: Audit a Python project's toolchain config against first-class conventions (uv, ruff, pyright strict, pytest+cov, SHA-pinned CI, Dependabot). Read-only.
---

Audit a Python project's configuration against the `/bootstrap-python` baseline. **Read-only — make no changes.** Stays at the config layer; does not run `uv sync` or invoke the tools (those have side effects: creating `.venv`, downloading packages).

**Target**: `$ARGUMENTS` if a path is given; otherwise the current working directory.

Run independent reads in parallel via multiple tool uses in one message.

## Step 0 — Preflight

Run these checks in parallel before inventorying.

**Required:**

- `pyproject.toml` exists at the target. If not, stop and tell the user this isn't a Python project the audit can run against — suggest `/bootstrap-python` to scaffold one. (No tooling is *strictly* required beyond file reads, since the audit only inspects config — but the soft checks below shape the report.)

**Soft checks (record in the report header; do not gate):**

- `uv --version` — note its absence so the user knows the toolchain isn't installed locally. Suggest `brew install uv` (macOS) or https://docs.astral.sh/uv/getting-started/installation/.
- `git --version` and whether the target is inside a git work tree (`git rev-parse --is-inside-work-tree`). If git is missing or the directory isn't a repo, the "lockfile committed?" check in §4 falls back to "uv.lock present on disk only — not tracked" instead of running `git ls-files`.
- `python3 --version` — not required (uv manages the interpreter), but useful context if uv is also missing.

If `uv` and/or `git` are missing, the audit still completes — it just reports lower-fidelity facts for the affected sections. Surface the degraded checks explicitly in the output so the reader knows what wasn't verifiable.

## What to inventory

### 1. `pyproject.toml`

Parse and report:

- **`[build-system]`** — backend (`hatchling` is baseline; `setuptools`/`poetry-core`/`flit` are deltas worth surfacing)
- **`[project]`** — `name`, `requires-python` (baseline: series-pin form like `"==3.14.*"`), presence of `[project.dependencies]` and `[project.optional-dependencies]`
- **`[dependency-groups].dev`** — must include `pyright`, `pytest`, `pytest-cov`, `ruff`. List what's missing.
- **`[tool.pyright]`** — `pythonVersion` and `typeCheckingMode` (must be `"strict"`)
- **`[tool.ruff]`** — `line-length`, `target-version`
- **`[tool.ruff.lint].select`** — must include the baseline families: `E`, `W`, `F`, `I`, `UP`, `B`, `SIM`, `C4`, `S`, `RUF`. List which baseline families are missing.
- **`[tool.ruff.lint.per-file-ignores]`** — `"tests/**"` should waive `S101`
- **`[tool.ruff.lint.isort].known-first-party`** — should list the package
- **`[tool.pytest.ini_options]`** — `addopts` includes `--cov=<package>` and `--cov-report=term-missing`; `testpaths = ["tests"]`
- **`[tool.coverage.run]`** — `branch = true`, `source` set to `src/<package>`
- **`[tool.coverage.report].exclude_also`** — at minimum `"if TYPE_CHECKING:"` and `"raise NotImplementedError"`

### 2. Python version coherence

Read each of these and confirm they all match on major.minor (this is the most common drift):

- `[project].requires-python` (baseline: series-pin form `"==X.Y.*"`)
- `[tool.pyright].pythonVersion`
- `[tool.ruff].target-version`

Flag every mismatch — even one out-of-sync entry produces silent type/lint behavior changes. Also flag if `requires-python` is unbounded (`">=X.Y"` rather than `"==X.Y.*"`) — the series-pin form is the convention.

If `.python-version` is present, flag it as a gap with a "drop in favor of `requires-python`" recommendation, unless it pins a more-specific patch version (e.g. `3.14.3`) that `requires-python` can't express. `requires-python` is the single source of truth; uv reads it for interpreter resolution.

### 3. Layout

- `src/<package>/` (baseline) vs flat `<package>/` (delta — note but don't gate on it; layout migrations are out of scope for the audit)
- `src/<package>/__init__.py` exists
- `src/<package>/py.typed` exists (required for downstream consumers to honor types)
- `tests/` directory exists with `tests/__init__.py`

### 4. Lockfile

- `uv.lock` present and committed (check `git ls-files uv.lock`)
- Out-of-date check is *not* performed here (would require `uv lock --check`, which writes to `.venv` on some configurations); just report presence.

### 5. Toolchain classification

Synthesize the markers below into a single classification that goes in the report header. This is the most load-bearing fact in the audit — it determines whether the project needs a migration or just patches.

- **uv-native** — `[build-system]` uses `hatchling` (or another PEP 517 backend uv supports), `[project]` populated, `uv.lock` present, no legacy markers
- **PEP 621 (non-uv)** — `[project]` populated but no `uv.lock`; possibly hatch / pdm / flit lockfiles instead
- **Poetry** — `[tool.poetry]` table present in `pyproject.toml`, usually with `poetry.lock`
- **Pipenv** — `Pipfile` and/or `Pipfile.lock` present
- **setup.py / setup.cfg** — legacy setuptools project. Sub-classify:
  - **declarative** — `setup.py` only contains a `setup(...)` kwargs call (or all metadata is in `setup.cfg`)
  - **with code** — `setup.py` contains imports, conditionals, dynamic version, custom build steps. Migration is manual.
- **pip + requirements** — `requirements*.txt` present, no `[project]` table, no other lockfile
- **mixed / unclear** — multiple of the above (e.g. Poetry pyproject + leftover requirements.txt + legacy setup.py)

**Legacy markers to inventory** (list each found; bucket them under the classification above):

- `requirements.txt`, `requirements-dev.txt`, `dev-requirements.txt`
- `setup.py` (capture: declarative vs. contains-code), `setup.cfg`
- `Pipfile`, `Pipfile.lock`
- `poetry.lock`, `[tool.poetry]` / `[tool.poetry.*]` tables
- `.flake8`, `.isort.cfg`, `[tool.black]` / `[tool.isort]` / `[tool.flake8]` in `pyproject.toml`

Linter leftovers (black/flake8/isort) are independent of the dependency-management toolchain — record them separately so the gap list can flag them even on an otherwise-uv-native project.

### 6. `.gitignore`

Should contain at least: `__pycache__/`, `*.py[oc]`, `build/`, `dist/`, `*.egg-info/`, `.venv`, `.coverage`, `htmlcov/`. List missing entries.

### 7. `.vscode/settings.json`

Should set:
- `python.analysis.typeCheckingMode = "strict"`
- `python.analysis.importFormat = "absolute"`
- `[python].editor.defaultFormatter = "charliermarsh.ruff"`
- `[python].editor.formatOnSave = true`
- `[python].editor.codeActionsOnSave` includes `source.fixAll.ruff` and `source.organizeImports.ruff`

### 8. `.github/workflows/`

For each `.yml` file:
- List job keys and `name:` values
- Check that **every** `uses:` is pinned by 40-char SHA (not a tag like `@v4` or `@main`). List any that aren't.
- Confirm a CI workflow exists with `lint` (ruff), `typecheck` (pyright), and `test` (pytest) jobs — by name or by what they run. Missing jobs are gaps.
- Confirm the workflow uses `astral-sh/setup-uv` and runs `uv lock --check` before `uv sync --dev`.

### 9. `.github/dependabot.yml`

- Exists?
- Has a `pip` ecosystem entry (Dependabot reads `pyproject.toml` for uv projects via the `pip` ecosystem)?
- Has a `github-actions` ecosystem entry?
- Minor/patch grouping configured for the Python ecosystem?

### 10. `CLAUDE.md`

- Exists at repo root?
- Contains a Python Toolchain section (or equivalent) covering: uv usage, pyright strict, ruff, pytest with coverage, SHA-pinned actions?

The exact wording from `/bootstrap-python` is one valid form — don't gate on text match. Gate on whether each topic is covered.

## Output

A single markdown report under ~500 words:

1. **Header** — project path, package name (from `[project].name` or equivalent), Python version pin, **toolchain classification** (`uv-native`, `Poetry`, `Pipenv`, `setup.py (declarative)`, `setup.py (contains code)`, `setup.cfg`, `pip + requirements`, `PEP 621 (non-uv)`, `mixed`), and a one-line summary (e.g. "Poetry project — full migration required" or "uv-native; 2 baseline rule families missing, 1 unpinned action")
2. **`pyproject.toml`** — table of sections vs. baseline with ✅/❌. For ruff `select`, list missing families explicitly. For non-uv classifications, mark uv-specific sections as "N/A until migrated".
3. **Python version coherence** — table of the four locations and their values; ❌ on any mismatch
4. **Layout & files** — `src/`, `py.typed`, `tests/__init__.py`, `.gitignore` entries, `.vscode/settings.json` keys
5. **Lockfile** — `uv.lock` present / committed (or `poetry.lock` / `Pipfile.lock` for non-uv classifications)
6. **Legacy toolchain** — list every marker found, grouped by source toolchain (Poetry / Pipenv / setup.py / setup.cfg / pip / linters). For `setup.py`, note declarative vs. contains-code.
7. **CI** — workflows, job names, SHA-pin status (call out unpinned actions explicitly with file:line), presence of `uv lock --check`
8. **Dependabot** — ecosystems present, grouping
9. **CLAUDE.md** — Python toolchain block present
10. **Gaps** — bullet list of concrete deltas vs. baseline, ordered by severity. Each gap names the exact thing wrong (e.g. "`[tool.ruff.lint].select` missing `S` (bandit) and `SIM`", "`actions/checkout@v4` in `ci.yml:12` not pinned by SHA", "`requires-python = ">=3.14"` is unbounded — convention is series-pin `"==3.14.*"`", "`.python-version` present but redundant with `requires-python`"). For non-uv classifications, **lead with the migration recommendation** (e.g. "Poetry project — `/bootstrap-python` will migrate `[tool.poetry]` deps to PEP 621 `[project]` and replace `poetry.lock` with `uv.lock`. Migration is the prerequisite for the other gaps to be addressable."). For `setup.py (contains code)`, flag that migration is **manual** — `/bootstrap-python` will refuse to auto-migrate and the user needs to extract metadata themselves first.

End with: "Run `/bootstrap-python` to apply the baseline (idempotent — safe on existing projects, works on a branch). For non-uv projects, it will run the migration first."

Facts and gaps only. No generic best-practice advice — the reader is the owner.
