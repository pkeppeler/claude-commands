# claude-commands

Personal collection of [Claude Code](https://docs.claude.com/en/docs/claude-code) custom slash commands. Markdown files I've iterated on and want to keep version-controlled.

## Layout

```
claude-commands/
├── README.md
├── CLAUDE.md          # context for Claude when editing commands in this repo
└── commands/
    └── <name>.md      # each file becomes /<name> in Claude Code
```

## How custom slash commands work

Claude Code looks for user-level slash commands in `~/.claude/commands/`. Each `.md` file in that directory becomes a command:

- File `~/.claude/commands/foo.md` → `/foo` in any Claude Code session.
- The file's body is the prompt that runs when you invoke the command.
- The string `$ARGUMENTS` in the body is replaced with whatever you type after the command name.
- Optional YAML frontmatter sets metadata:

  ```markdown
  ---
  description: One-line summary shown in the slash-command picker
  ---

  Body of the prompt goes here. $ARGUMENTS will be substituted.
  ```

## Installation

Symlink the commands you want into `~/.claude/commands/`. Symlinks (rather than copies) mean `git pull` updates take effect immediately.

**Install one command:**

```bash
mkdir -p ~/.claude/commands
ln -sf "$PWD/commands/bootstrap-python.md" ~/.claude/commands/bootstrap-python.md
```

**Install all commands in this repo:**

```bash
mkdir -p ~/.claude/commands
for f in commands/*.md; do
  ln -sf "$PWD/$f" ~/.claude/commands/"$(basename "$f")"
done
```

**Uninstall a command:** delete the symlink (`rm ~/.claude/commands/<name>.md`). The file in this repo is untouched.

## Available commands

| Command | What it does |
|---|---|
| `/bootstrap-python` | Scaffold a fresh Python repo with first-class conventions: uv, ruff, pyright strict, pytest with coverage, GitHub Actions CI (SHA-pinned), Dependabot. Pass the package name as an argument: `/bootstrap-python my_package`. |
| `/bootstrap-github-repo` | Apply first-class GitHub repo configuration via API: squash-only merging, auto-merge + update-branch, security toggles (vulnerability alerts, Dependabot security updates, secret scanning, push protection, Private Vulnerability Reporting), a default-branch ruleset (require PR + CODEOWNERS review, required status checks, block force pushes/deletions, linear history) with the repo owner as bypass actor, and baseline files (`CODEOWNERS`, `SECURITY.md`, `dependabot.yml`). Plan-aware (skips features unavailable on private + free). Idempotent. |
| `/audit-github-repo` | Read-only inventory of a GitHub repo's settings, security features, rulesets, branch protection, and convention files, with a gaps summary against the first-class baseline. Pairs with `/bootstrap-github-repo` — run audit, then bootstrap, then audit again. |

## Authoring a new command

1. Create `commands/<name>.md`.
2. Add frontmatter with a `description` (shown in the slash picker).
3. Write the body as instructions to Claude. Use `$ARGUMENTS` for user-supplied input.
4. Symlink it into `~/.claude/commands/` (see above).
5. Test by running `/<name>` in a Claude Code session.

Aim for prompts that read like clear instructions to a careful colleague: state the goal, list the steps, name the verification. Avoid vague directives ("set things up nicely") — Claude will fill the gaps in ways you didn't intend.
