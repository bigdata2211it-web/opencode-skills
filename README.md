# opencode-skills → ai-cli-skills

Collection of AI agent skills/instructions, packaged for **multiple CLI/IDE tools**:
opencode, Claude Code, Cursor, Codex CLI, and Cline. Same content, different wrappers.

## Skills

| Skill | Description |
|-------|-------------|
| **mybd-cryfs** | Encrypted credentials vault with cryfs (FUSE). Cross-platform: Linux, macOS, Windows. Covers install, migration, daily open/close, auto-lock, agent discipline, backup, password change, troubleshooting. |

## Pick your tool

| Tool | Path in this repo | Install target | Format |
|------|-------------------|----------------|--------|
| [opencode](https://opencode.ai) | `opencode/mybd-cryfs/SKILL.md` | `~/.config/opencode/skills/mybd-cryfs/SKILL.md` | frontmatter `name`/`description` |
| [Claude Code](https://claude.com/claude-code) | `claude-code/mybd-cryfs/SKILL.md` | `~/.claude/skills/mybd-cryfs/SKILL.md` | same frontmatter, skills dir |
| [Cursor](https://cursor.com) | `cursor/mybd-cryfs/mybd-cryfs.mdc` | `.cursor/rules/mybd-cryfs.mdc` | `.mdc` with `description`/`globs`/`alwaysApply` |
| [Codex CLI](https://github.com/openai/codex) | `codex/mybd-cryfs/AGENTS.md` | `AGENTS.md` at project root (or `~/.codex/AGENTS.md`) | plain markdown |
| [Cline](https://cline.bot) | `cline/mybd-cryfs/mybd-cryfs.md` | `.clinerules/mybd-cryfs.md` | plain markdown in `.clinerules/` |

Agnostic body (no wrapper, for porting to other tools) lives at `_content/mybd-cryfs.md`.

## Install

### opencode

```bash
cp -r opencode/mybd-cryfs ~/.config/opencode/skills/
```
Restart opencode. Or register the repo as a skill source:
```json
{ "skills": { "urls": ["https://github.com/bigdata2211it-web/opencode-skills"] } }
```

### Claude Code

```bash
mkdir -p ~/.claude/skills
cp -r claude-code/mybd-cryfs ~/.claude/skills/
```
Restart Claude Code. The skill is auto-discovered from `~/.claude/skills/`.

### Cursor

Copy the `.mdc` file into your project's rules folder:
```bash
mkdir -p .cursor/rules
cp cursor/mybd-cryfs/mybd-cryfs.mdc .cursor/rules/
```
`alwaysApply: false` means it triggers on keyword matches / manual `@`-mention.
Set `alwaysApply: true` if you want it in every request.

### Codex CLI

Drop `AGENTS.md` at the project root (or merge into an existing one):
```bash
cp codex/mybd-cryfs/AGENTS.md ./AGENTS.md
```
For global use, place at `~/.codex/AGENTS.md`.

### Cline

Place the rule file in the project's `.clinerules/` folder:
```bash
mkdir -p .clinerules
cp cline/mybd-cryfs/mybd-cryfs.md .clinerules/
```
Cline auto-loads everything under `.clinerules/`.

## Skill format (per tool)

### opencode / Claude Code
```
skill-name/
  SKILL.md    # frontmatter (name, description) + markdown body
```

### Cursor
```
.cursor/rules/
  skill-name.mdc   # frontmatter (description, globs, alwaysApply) + markdown body
```

### Codex CLI
```
AGENTS.md           # plain markdown, instructions for the Codex agent
```

### Cline
```
.clinerules/
  skill-name.md     # plain markdown, auto-loaded
```

## Porting to another tool

Take `_content/mybd-cryfs.md` (no frontmatter) and wrap it in your tool's format.
Open an issue/PR if you add a new wrapper.

## Verify the vault (after setup)

- Closed: mountpoint is not a mountpoint, `ls` is empty.
- Backend dir contains hex-named subdirs + `cryfs.config`.
- `file <backend>/<hex>/*` reports `data` (binary, unreadable).
- Open: mountpoint is a mountpoint, file count matches original.
- Password is not in `~/.bash_history`, not in any config, not in any doc.
- `mybd-close` succeeds after `cd ~`.
- Auto-lock fires after N idle minutes.
- External backup of the backend exists (or the user has explicitly accepted the no-backup risk).

## License

MIT
