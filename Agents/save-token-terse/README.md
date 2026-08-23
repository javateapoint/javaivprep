# Terse Mode

A small, portable "answer in fewer words" ruleset for this project — same
idea as the community `caveman` skill, scaled down to just the part that
matters for a team on IntelliJ + GitHub Copilot: less output prose, code
and commands always exact, intensity adjustable.

## Files

- `.github/copilot-instructions.md` — the actual ruleset. GitHub Copilot
  (VS Code, github.com, and the JetBrains/IntelliJ Copilot plugin) reads
  this automatically from every repo it's enabled in — no install step.
- `TERSE-SKILL.md` — the same ruleset packaged as a Claude-style skill,
  for Claude Code / Claude.ai / any skills-compatible agent, with a
  `/terse light|medium|extreme|off` trigger.

## Setup

### GitHub Copilot (VS Code or github.com)
Nothing to do — drop `.github/copilot-instructions.md` into the repo root
and Copilot Chat picks it up on the next request in that workspace.

### GitHub Copilot in IntelliJ / JetBrains
Recent versions of the Copilot plugin also read `.github/copilot-instructions.md`
from the project root. If your plugin version doesn't yet, paste the file's
contents into the plugin's custom-instructions setting if one exists, or copy
it into your prompt as needed.

### IntelliJ's native AI Assistant (not Copilot)
This one does not auto-read repo files. Go to
**Settings → Tools → AI Assistant → Custom instructions** and paste in the
contents of `.github/copilot-instructions.md`.

### Claude Code / Claude.ai
```
npx skills add <this-repo-or-path> --skill terse-mode
```
or just copy `TERSE-SKILL.md` into your skills folder. Trigger with
`/terse`, `/terse extreme`, etc.

## Changing the intensity

Edit one line at the top of `.github/copilot-instructions.md`:

```
TERSE_LEVEL: MEDIUM
```

to `LIGHT`, `MEDIUM`, or `EXTREME`. Every tool reading that file picks up
the change on its next request — no reinstall.

## Honest limits (read before you rely on the token savings)

- This only shrinks **output** tokens — your assistant's prose. It does
  nothing to the input side (files, context, tool results), unlike
  caveman's newer proxy component.
- On tasks that were already going to get a short answer, savings are
  small or zero.
- Nothing here is measured or benchmarked for this repo specifically —
  it's a style instruction, not a compression engine. If you want provable
  token/cost numbers, you'd need to log before/after token counts yourself
  or adopt something like caveman's proxy, which does real byte-level
  compression with recovery.
