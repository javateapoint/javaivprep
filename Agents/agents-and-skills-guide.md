# Building Agents & Skills — End-to-End Guide

*A reference for building custom AI agents and skills for our project, using GitHub Copilot (IntelliJ) and Google Antigravity.*

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [How the Agent Loop Actually Works](#2-how-the-agent-loop-actually-works)
3. [Skill Anatomy](#3-skill-anatomy)
4. [GitHub Copilot (IntelliJ) — Setup & Usage](#4-github-copilot-intellij--setup--usage)
5. [Google Antigravity — Setup & Usage](#5-google-antigravity--setup--usage)
6. [Beginner → Advanced Roadmap](#6-beginner--advanced-roadmap)
7. [Writing Good Skills — Rules & Anti-Patterns](#7-writing-good-skills--rules--anti-patterns)
8. [Worked Example: A Real Skill](#8-worked-example-a-real-skill)
9. [Checklist Before Scaling Up](#9-checklist-before-scaling-up)
10. [Glossary](#10-glossary)

---

## 1. Core Concepts

Two different things get called "agent" and "skill" — keep them separate in your head.

| Concept | What it is | Analogy |
|---|---|---|
| **Agent** | A *mode of operation*: model reasons → calls a tool → observes result → repeats until the task is done | An employee who can take actions, not just give advice |
| **Skill** | A *packet of instructions* the agent loads when relevant | A laminated SOP card the employee pulls out for a specific task |

The agent is the loop. The skill is food for the loop. You don't "install" an agent so much as put a model into agent mode with tools; you build skills as files that get loaded into that agent's context on demand.

---

## 2. How the Agent Loop Actually Works

Every coding agent — Copilot Agent Mode, Antigravity, Claude Code, etc. — runs the same underlying cycle:

1. **Instructions** — a system prompt / instructions file shapes overall behavior
2. **Perceive** — the model reads your request + current file/project state
3. **Plan** — it decides what to do (and may write out a plan first)
4. **Act** — it calls a tool: edit a file, run a terminal command, call an MCP server, search docs
5. **Observe** — the tool's result (diff, terminal output, error) goes back into context
6. **Repeat** until the task is complete or it needs your input

**Skills plug into step 1 and step 3** — they don't add new abilities, they add *the right instructions at the right time* so the model doesn't need you to explain project conventions in every prompt.

Why not just load everything all the time? Because agents get **"tool bloat"** / **"context rot"** — dumping 40–50k tokens of unused instructions into every conversation makes responses slower, more expensive, and more likely to get confused by irrelevant rules. The fix both Copilot and Antigravity use is **progressive disclosure**:

- At the start of a session, the agent only sees a short **menu**: skill names + one-line descriptions
- It reads the **full skill content** only when your request matches a description
- Once the task is done, that skill's detailed context can be released

---

## 3. Skill Anatomy

A skill is a folder. The only required file is `SKILL.md`:

```
my-skill/
├── SKILL.md          # required — description + instructions
├── scripts/           # optional — helper scripts the agent can run
├── examples/          # optional — reference implementations
└── resources/         # optional — templates, boilerplate, checklists
```

Minimal `SKILL.md` template:

```markdown
---
name: short-kebab-case-name
description: One or two sentences. This is the ONLY thing the agent sees before deciding to load the rest — make it specific enough to trigger correctly, e.g. "Use when creating or reviewing database migrations. Covers naming convention, rollback rules, and required checklist."
---

# Skill Title

## When to use this
Concrete trigger scenarios — be specific, not generic.

## Steps / Instructions
1. Step one
2. Step two
3. ...

## Project-specific gotchas
Anything learned the hard way — past incidents, quirks of our staging
environment, things that look right but break our pipeline.

## Reference
Links to internal docs, related files, or scripts in this skill folder.
```

---

## 4. GitHub Copilot (IntelliJ) — Setup & Usage

### 4.1 Always-on project instructions
Create one of these in your repo root (both are supported; `AGENTS.md` is the emerging cross-tool standard):

```
.github/copilot-instructions.md
AGENTS.md
```

Put here: coding conventions, build/test commands, review checklist, architecture notes. This loads on *every* Copilot chat interaction in this repo — no discovery step needed.

You can also scope narrower instruction files, e.g. commit message style:

```
.github/git-commit-instructions.md
```

### 4.2 Enabling Agent Skills (preview)
`Settings → GitHub Copilot → Chat → Agent → Enable Agent Skills`

> Note: if you're on Copilot Business/Enterprise, an admin may need to enable the "Editor preview features" policy first.

Place skills in your repo so the team shares them. Community examples exist at `github/awesome-copilot` and `anthropics/skills` if you want references before writing your own.

### 4.3 Custom Agents
A custom agent is a config describing: name, description, instructions, knowledge sources, and **tool policy** (what it's allowed to touch).

Workflow:
1. Generate a starter config via the custom agents quickstart
2. Fill in name / description / instructions / allowed tools
3. Select it from the agent picker in Copilot Chat
4. Test it, refine instructions based on real usage

Good first custom agents to build:
- **Code Reviewer** — read-only tools, focused on your review checklist
- **Release Manager** — access limited to deploy scripts and changelog files

### 4.4 Sub-agents
Sub-agents let a main agent delegate an isolated side-task (e.g. "also update the changelog") without polluting the main task's context window. Useful once your primary agent handles multi-part requests regularly.

### 4.5 MCP (Model Context Protocol)
MCP servers connect the agent to real systems — issue trackers, internal APIs, databases — instead of just following written instructions. Configure at:

```
Settings → Tools → GitHub Copilot → Chat → MCP Server and Tool Auto-approve Configuration
```

You can auto-approve specific servers/tools to reduce interruption during agent runs — do this only for tools you trust with write access.

---

## 5. Google Antigravity — Setup & Usage

### 5.1 Skill scopes

| Scope | Location | Use for |
|---|---|---|
| **Project** | `<project-root>/.agents/skills/<name>/SKILL.md` | Project-specific scripts, deployment steps, proprietary framework conventions |
| **Global** | `~/.gemini/config/skills/` | Personal utilities usable across all your projects (formatters, generators, personal workflows) |

### 5.2 Discovery
Fully automatic — no need to invoke a skill by name. At session start, the agent sees the list of skill names + descriptions; if your request matches one, it reads the full `SKILL.md`. This is why the **description field quality matters more than anything else** in the file.

### 5.3 Authoring workflow
1. Pick a task you repeat often
2. Write the `SKILL.md` (see [Section 3](#3-skill-anatomy))
3. Test with a natural request that *doesn't* name the skill explicitly
4. If Antigravity does the task but imperfectly, let the agent itself fix and **update the SKILL.md** to encode the correction — this compounds over time
5. Commit the skill folder alongside your code

### 5.4 Sub-agents / Agent Manager
In Antigravity's model, the main Agent Manager acts like a project manager, delegating sub-tasks to specialized sub-agents defined by their skills — useful once you have a small library of skills covering distinct responsibilities (testing, migrations, docs generation, etc.).

---

## 6. Beginner → Advanced Roadmap

1. **Start with instructions, not agents.** Add `.github/copilot-instructions.md` / `AGENTS.md` with conventions and build commands. No skill machinery needed yet.
2. **Turn one recurring task into a skill.** Pick something you explain the same way every week (e.g. "write a migration," "scaffold a REST endpoint"). Write it once as `SKILL.md`.
3. **Test discovery, not just correctness.** Ask a vague, natural question that *should* trigger the skill without naming it. If it doesn't fire, your description is too generic.
4. **Split instead of expanding.** Once a skill covers more than one job, break it into two. Keeps context lean and avoids confusing the agent about which rules apply.
5. **Introduce a custom agent** once you have 3–4 skills — give it a narrow job and a restricted tool policy.
6. **Delegate with sub-agents** for isolated side-tasks within a larger request.
7. **Connect real tools via MCP** once skills feel solid — this is the "advanced" tier where the agent can act on live systems, not just follow written instructions.

---

## 7. Writing Good Skills — Rules & Anti-Patterns

**Do:**
- Make the description read like a search query: *"Use when reviewing PRs for security issues"*
- Encode things the model doesn't already know — your naming conventions, past incidents, environment quirks
- Keep one skill = one job
- Version-control skills like code; review changes via normal PRs
- Scope tool/write access tightly for anything agentic (read-only reviewers vs. deploy-capable agents)

**Avoid:**
- Vague descriptions ("helps with database stuff") — the agent can't match intent to a vague trigger
- Mega-skills covering testing + deployment + linting + docs in one file (tool/context bloat)
- Teaching the model generic knowledge it already has — spend the tokens on what's project-specific
- Giving broad tool access to a skill/agent that only needs to read and suggest

---

## 8. Worked Example: A Real Skill

```markdown
---
name: db-migration
description: Use when creating or reviewing database migrations for our project. Covers naming convention, rollback rules, and required review checklist.
---

# DB Migration Skill

## When to use this
Any time a schema change is requested — new table, column, index, or constraint.

## Steps
1. Check the latest migration number in /migrations
2. Name the file: V{next_number}__{snake_case_description}.sql
3. Every migration must include a corresponding rollback script
4. Never DROP a column without a two-step deprecate → remove process
5. Run `./scripts/validate_migration.sh` before committing

## Project-specific gotchas
- Our staging DB doesn't support certain constraint types — test there first
- CI blocks merges if a migration lacks a rollback script
```

---

## 9. Checklist Before Scaling Up

- [ ] Every skill description is specific enough to act as a trigger
- [ ] Each skill does exactly one job
- [ ] Instructions capture team-specific knowledge, not generic facts
- [ ] Tool/write access is scoped per agent (least privilege)
- [ ] Skills and agent configs live in version control
- [ ] At least one teammate has reviewed each skill file, same as code

---

## 10. Glossary

- **Agent** — a model operating in a perceive → plan → act → observe loop with tool access
- **Agent mode** — the mode (vs. plain autocomplete/chat) where the model can take actions
- **Skill** — a folder (minimally a `SKILL.md`) of instructions loaded on demand
- **Progressive disclosure** — loading only skill summaries by default, full content on match
- **Sub-agent** — a secondary agent a main agent delegates an isolated task to
- **MCP (Model Context Protocol)** — a standard for connecting agents to external tools/systems (APIs, databases, trackers)
- **Custom agent** — a named, configured agent with its own instructions, knowledge sources, and tool policy
- **AGENTS.md** — an emerging cross-tool standard file for project-wide agent instructions

---

*Last updated: keep this file current — when a skill teaches the agent something new the hard way, update this guide too.*
