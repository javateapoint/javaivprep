---
name: terse-mode
description: Compress response prose to save output tokens while keeping code, commands, file paths, and technical details byte-for-byte exact. Use whenever the user says "terse", "be brief", "/terse", asks to save tokens, or has set a terseness preference for this project. Supports three intensities (light/medium/extreme) set via /terse <level> or the TERSE_LEVEL value in .github/copilot-instructions.md.
---

# Terse Mode

Trims response prose. Never trims code, commands, paths, flags, config
values, error text, or version numbers — those are always reproduced exact.

## Trigger

- `/terse` — toggle on at the current default level (medium, unless changed)
- `/terse light|medium|extreme` — set intensity
- `/terse off` or "normal mode" — disable for the rest of the conversation
- If `.github/copilot-instructions.md` exists in the project and sets
  `TERSE_LEVEL`, use that as the default level without being asked.

## Hard rules (all levels)

1. Code, commands, file paths, flags, config keys, variable names, error
   messages, and version numbers are always byte-for-byte exact. Terseness
   only touches the sentences around them.
2. Never drop information that changes correctness or safety. Compress
   wording, not content.
3. If a correct, complete answer needs more words than the level allows,
   use the words anyway. This is a style constraint, not permission to be
   wrong, incomplete, or to skip reasoning steps.
4. Multi-step debugging or reasoning still happens in full — only how it's
   *reported* gets shorter.

## Levels

**light** — cut filler and hedges ("I think", "it's worth noting"). Normal
sentences, normal tone. ~10–20% shorter than an unprompted answer.

**medium** — bullets over paragraphs, short sentences, no restating the
question, one-sentence explanations unless the "why" needs more.

**extreme** — telegraphic. Drop articles and connective words where meaning
survives. Fragments over sentences. Bluntest, densest option.

## Example (same content, three levels)

- light: "Test fails because the mock isn't reset between cases — a common
  gotcha."
- medium: "Mock not reset between tests → stale state → fails. Reset in
  `afterEach`."
- extreme: "Mock not reset btwn tests. Fix: reset in `afterEach`."

## Don't

- Don't compress anything the user pasted (logs, stack traces, code).
- Don't apply this to legal, compliance, or security explanations — write
  those in full regardless of level.
- Don't let terseness become vagueness. A shorter wrong answer is worse
  than a longer right one.
