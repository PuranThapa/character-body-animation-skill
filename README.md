# Character Body Animation — a Claude Code skill

Makes Claude animate code-drawn characters (canvas, SVG, WebGL) like cartoon characters instead of robots: bending elbows, connected shoulders, natural walk cycles, sitting and gestures.

## Before vs after

<video src="https://github.com/PuranThapa/character-body-animation-skill/raw/main/demo-before-vs-after.mp4" controls muted width="100%"></video>

[Watch the before vs after demo](demo-before-vs-after.mp4)

## Install

### Option 1: plugin (recommended)
Inside Claude Code:
```
/plugin marketplace add PuranThapa/character-body-animation-skill
/plugin install character-body-animation@character-body-animation
```

### Option 2: copy the skill manually
```bash
git clone https://github.com/PuranThapa/character-body-animation-skill.git
mkdir -p ~/.claude/skills
cp -r character-body-animation-skill/skills/character-body-animation ~/.claude/skills/
```
Restart Claude Code. To use it in one project only, copy the folder to `.claude/skills/` inside that project instead.

### Other agents (Codex, Cursor, Copilot, Gemini CLI…)
`SKILL.md` follows the open Agent Skills format, so it works outside Claude Code too. Copy the `skills/character-body-animation` folder into your tool's skills directory (check its docs for the exact path, e.g. `.agents/skills/` or `~/.codex/skills/` for Codex). The `.claude-plugin/` folder is only for Claude Code and can be ignored.

If your agent doesn't support skills, reference the file from `AGENTS.md` (or your tool's instructions file):
```
When animating characters, follow the rules in skills/character-body-animation/SKILL.md
```

> The skill was written and tested with Claude; other models can follow the same rules, but results may vary.

## Use
Just ask Claude to animate a character, e.g. *"make a canvas animation of a boy walking up to a girl and waving"*. Claude loads the skill automatically when the task matches.

## License
MIT
