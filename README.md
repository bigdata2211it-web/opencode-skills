# opencode-skills

Collection of [opencode](https://opencode.ai) skills.

## Skills

| Skill | Description |
|-------|-------------|
| [mybd-cryfs](mybd-cryfs/SKILL.md) | Encrypted credentials vault with cryfs (FUSE). Cross-platform: Linux, macOS, Windows. Covers install, migration, daily open/close, auto-lock, agent discipline, backup, troubleshooting. |

## Usage

### Install a skill

Copy the skill folder into your opencode skills directory:

```bash
# Global skills directory
cp -r mybd-cryfs ~/.config/opencode/skills/

# Or project-level
cp -r mybd-cryfs .opencode/skills/
```

Restart opencode for the skill to be loaded.

### Register skills from this repo

Add to your `opencode.json`:

```json
{
  "skills": {
    "urls": ["https://github.com/bigdata2211it-web/opencode-skills"]
  }
}
```

Or clone and point to the local path:

```json
{
  "skills": {
    "paths": ["~/opencode-skills"]
  }
}
```

## Skill format

Each skill lives in its own folder with a `SKILL.md` file:

```
skill-name/
  SKILL.md    # frontmatter (name, description) + instructions
```

See the [opencode docs](https://opencode.ai) for the full skill specification.

## License

MIT
