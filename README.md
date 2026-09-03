# zach-skills

The [Claude Code](https://claude.com/claude-code) skills I actually use.

A skill is a directory plus a `SKILL.md` — the `name` and `description` in its YAML frontmatter are what decide when it fires. This repo groups them **into layers**, one directory per layer, with every skill inside self-contained.

## Layers

| Layer | What it's for | Skills |
| --- | --- | --- |
| [`herdr/`](herdr/) | Orchestrating parallel agents inside the [Herdr](https://herdr.dev) terminal multiplexer | `herdr-dispatch`, `herdr-monitor` |
| [`normal/`](normal/) | Everyday skills that need no external tool — currently two registers for explaining things | `explaining-mechanisms-progressively`, `human-words` |

## Install

Claude Code loads user-level skills from `~/.claude/skills/<name>/SKILL.md`, and the directory name *is* the skill name. Symlink them in — edits take effect immediately:

```bash
git clone https://github.com/wine-fall/zach-skills.git ~/src/zach-skills

# One skill
ln -s ~/src/zach-skills/herdr/herdr-dispatch ~/.claude/skills/herdr-dispatch

# A whole layer
for d in ~/src/zach-skills/herdr/*/; do
  ln -sfn "$d" ~/.claude/skills/"$(basename "$d")"
done
```

Once installed, call them directly as `/herdr-dispatch`, `/herdr-monitor`, or let Claude fire them on its own from the `description`.

## Conventions

- One skill per directory, always entered through `SKILL.md`; helper scripts and reference docs live alongside it.
- A `description` must spell out **when to trigger and when not to** — it is the only routing signal the model gets.
- The layer directories are for grouping only and do not affect loading: a skill's name comes from the directory you install it as, not from where it sits in this repo.
