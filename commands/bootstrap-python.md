---
description: Scaffold a new Python repo with first-class conventions (uv, ruff, pyright strict, pytest+cov, SHA-pinned CI, Dependabot)
---

Bootstrap a fresh Python repo with first-class conventions.

**Package name** (used as the importable name): `$ARGUMENTS`

If `$ARGUMENTS` is empty, derive the package name from the current directory (snake_case it), confirm with the user, then proceed. Use that name everywhere `<package>` appears below.

Create each file, then run `uv sync --dev` and verify all of `uv run pyright`, `uv run ruff check`, `uv run ruff format --check`, and `uv run pytest` pass on the empty package.

NOTE: Python 3.14 is current as of 2026 — bump to whatever is the latest stable release at the time you run this. Update `requires-python`, `.python-version`, `[tool.pyright].pythonVersion`, and `[tool.ruff].target-version` together.

---

## 1. `pyproject.toml`

- Python `>=3.14`, `hatchling` build backend, `src/` layout
- `[dependency-groups].dev = ["pyright", "pytest", "pytest-cov", "ruff"]`
- `[tool.pyright]`: `pythonVersion = "3.14"`, `typeCheckingMode = "strict"`
- `[tool.ruff]`:
  - `line-length = 100`
  - `target-version = "py314"`
- `[tool.ruff.lint]`:
  ```toml
  select = [
    "E", "W",   # pycodestyle
    "F",        # pyflakes (real bugs)
    "I",        # isort (import sorting)
    "UP",       # pyupgrade (modernize syntax)
    "B",        # flake8-bugbear (likely bugs)
    "SIM",      # flake8-simplify
    "C4",       # flake8-comprehensions
    "S",        # flake8-bandit (security)
    "RUF",      # ruff-specific rules
  ]
  ```
- `[tool.ruff.lint.per-file-ignores]`:
  - `"tests/**" = ["S101"]` — asserts are fine in tests
- `[tool.ruff.lint.isort]`:
  - `known-first-party = ["<package>"]`
- `[tool.pytest.ini_options]`:
  - `addopts = "--cov=<package> --cov-report=term-missing"`
  - `testpaths = ["tests"]`
  - `xfail_strict = true` — a passing `xfail` is treated as a failure instead of a silent xpass; keeps stale xfail markers from accumulating
  - `filterwarnings = ["error"]` — turns warnings into test failures so deprecation/library warnings are addressed when they appear, not after they pile up
- `[tool.coverage.run]`:
  - `branch = true`
  - `source = ["src/<package>"]`
- `[tool.coverage.report]`:
  - `exclude_also = ["if TYPE_CHECKING:", "raise NotImplementedError"]`
  - No `fail_under` threshold yet — add one (e.g. 70) once the project has enough real tests for it to be meaningful.

## 2. `.python-version`

Single line: `3.14`

## 3. `.envrc`

Single line for direnv users: `source .venv/bin/activate`

## 4. `.gitignore`

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

Plus any project-specific secrets paths (ask the user).

## 5. `src/<package>/py.typed`

Empty marker file. Tells Pyright/Pylance to enforce types for downstream consumers of this package.

## 6. `src/<package>/__init__.py`

Leave minimal/empty. Add `__all__` later, only on modules that actually expose a public API.

## 7. `tests/__init__.py`

Empty file so pytest discovers the directory as a package.

## 8. `.vscode/settings.json`

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

## 9. `.github/workflows/ci.yml`

Three jobs (`lint`, `typecheck`, `test`) running in parallel on `pull_request` and `push` to `main`. Each job:

- `actions/checkout` (SHA-pinned)
- `astral-sh/setup-uv` (SHA-pinned) with `enable-cache: true`
- `uv lock --check`
- `uv sync --dev`
- Then the job-specific command:
  - `lint`: `uv run ruff check` then `uv run ruff format --check`
  - `typecheck`: `uv run pyright`
  - `test`: `uv run pytest`

**Pin every action by full 40-char SHA** with a `# vX.Y.Z` comment. Look up the current SHA for each action's latest release before writing the file — do not invent SHAs.

## 10. `.github/dependabot.yml`

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

Major bumps come as individual PRs; minor/patch are grouped to cut PR noise.

## 11. `CLAUDE.md`

Drop the conventions block below at the repo root, plus any project-specific rules the user names.

```markdown
## Python Toolchain

- Python 3.14 (pinned in `.python-version`)
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
- **Pydantic at I/O boundaries** — tool params/returns, config, external API responses, anything crossing a process or network boundary. Internal helpers can use `TypedDict`, dataclass, or plain dict where it fits.
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

## After scaffolding

1. Run `uv sync --dev` to materialize the venv and lockfile.
2. Run `uv run pyright`, `uv run ruff check`, `uv run ruff format --check`, `uv run pytest` and confirm all four exit clean.
3. Report back to the user: what was created, the four commands and their results, and any per-project rules to add to `CLAUDE.md`.
