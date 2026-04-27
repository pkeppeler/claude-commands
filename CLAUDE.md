# claude-commands

This repo holds personal Claude Code slash commands as markdown files. Each file in `commands/` is symlinked into `~/.claude/commands/` and becomes a `/<name>` command.

## File format

```markdown
---
description: One-line summary shown in the slash-command picker
---

Body of the prompt — instructions to Claude that run when the command is invoked.
$ARGUMENTS is replaced with whatever the user typed after the command name.
```

## When editing or creating commands here

- **Optimize for clarity to Claude.** The body is a prompt, not documentation. State the goal, list the steps, name the verification at the end.
- **Be explicit about placeholders.** If the command takes arguments, say up front how `$ARGUMENTS` is used. If it might be empty, specify the fallback (derive from context, ask the user).
- **Anchor non-obvious choices.** When a step bakes in a specific tool or convention (e.g. "use uv, not pip"), the command should briefly say *why* — otherwise Claude may "improve" it on the fly.
- **Keep it tight.** Long commands get skimmed. Group related steps; cut anything that's not load-bearing.
- **End with verification.** A good command tells Claude how to confirm the work succeeded (commands to run, files to check) and what to report back to the user.

## Repo conventions

- One command per file. Filename (without `.md`) is the slash command name.
- Use kebab-case for filenames (`bootstrap-python.md`, not `bootstrap_python.md` or `BootstrapPython.md`).
- Update the "Available commands" table in `README.md` when adding or removing commands.
