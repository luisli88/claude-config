# claude-config

Personal Claude Code configuration: global preferences, custom skills, and development constitution.

## Structure

```
~/.claude/
├── CLAUDE.md          # Global preferences applied to every Claude Code session
├── constitution.md    # Universal development principles (tech-agnostic)
├── settings.json      # Claude Code settings (statusline, plugins)
└── skills/            # Custom slash commands
    ├── api-route/
    ├── code-review/
    └── commit-pro/
```

## Setup

### New machine

```bash
# Backup existing config if any
mv ~/.claude ~/.claude.bak

# Clone repo
git clone git@github.com:luisli88/claude-config.git ~/.claude
```

### Link constitution in a project

The `constitution.md` defines universal development principles. Reference it from any project's `CLAUDE.md`:

```bash
# From the project root
ln -s ~/.claude/constitution.md ./constitution.md
```

Then add to the project's `CLAUDE.md`:

```markdown
## Constitution
⚠️ Read `./constitution.md` before any implementation task.
```

## Updating

Changes to `CLAUDE.md`, `constitution.md`, or skills take effect immediately in the next Claude Code session — no restart needed.

```bash
cd ~/.claude
git add <file>
git commit -m "feat(claude): ..."
git push
```
