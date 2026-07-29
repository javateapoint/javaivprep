# Building Custom AI Agents: Beginner to Pro
### GitHub Copilot Custom Agents · Claude Code Subagents · Framework-Agnostic LLM Agents · Token-Optimized Pro Agent

This guide covers three distinct ways senior engineers build "custom agents" today, since the term means different things depending on your tooling:

| Type | What it is | Where it runs |
|---|---|---|
| **GitHub Copilot custom agent** | A `.agent.md` persona file with its own system prompt, tool allowlist, and model | VS Code, JetBrains/IntelliJ, Copilot CLI, Copilot coding agent |
| **Claude Code subagent** | An isolated Claude instance with its own context window, prompt, and tools, spawned by a main session | Claude Code CLI (usable from IntelliJ's built-in terminal) |
| **Generic LLM agent** | A hand-rolled loop: system prompt + tool schemas + a model API, framework-agnostic | Anywhere — your own service, script, or backend |

Each part below goes beginner → intermediate → pro, with a working sample you build first and then enhance. Part D at the end covers the **token-optimized "pro" agent** — same accuracy, meaningfully lower cost.

---

# PART A — GitHub Copilot Custom Agents

## A.1 What it actually is

A custom agent is a Markdown file with YAML frontmatter (name, description, tools, model) followed by a plain-language system prompt. Copilot loads it as a selectable persona in the chat mode dropdown, or auto-delegates to it based on the `description` field. This feature was formerly called "custom chat modes" (`.chatmode.md`) and was renamed to **custom agents** (`.agent.md`) in late 2025. **Custom agents, sub-agents, and the plan agent reached general availability for JetBrains IDEs (including IntelliJ) in March 2026.**

## A.2 Beginner: Build Your First Sample Agent

**Goal**: a narrow, read-only "Test Writer" agent that only writes JUnit tests and never touches production code.

### Step 1 — Create the directory

```
your-repo/
└── .github/
    └── agents/
        └── test-writer.agent.md
```

### Step 2 — Write the agent file

```markdown
---
name: test-writer
description: >
  Writes and repairs JUnit 5 unit tests for Java/Spring Boot classes.
  Use for "write tests", "add test coverage", "fix failing test".
tools: ["read", "search"]
model: claude-sonnet-5
---

# Test Writer

You write JUnit 5 tests using Mockito for a Spring Boot codebase.

Rules:
- Never modify production (non-test) source files.
- Every public method gets at least one happy-path and one failure-path test.
- Use `@ExtendWith(MockitoExtension.class)`, not Spring context loading, unless
  the class under test genuinely needs Spring wiring (e.g. `@WebMvcTest`).
- Prefer `assertThatThrownBy` (AssertJ) over `assertThrows` for readability.
- If a class has no clear seam for mocking a dependency, say so and suggest a
  refactor rather than writing a brittle test.
```

### Step 3 — Test it

In VS Code or IntelliJ Copilot Chat: open the agent/mode dropdown, select **test-writer**, and ask: *"Write tests for OrderService.calculateTotal"*. Confirm it stays read-only and only proposes new test files.

## A.3 Enhance It (Beginner → Pro)

| Enhancement | How |
|---|---|
| **Restrict tools further** | Drop to `tools: ["read"]` only if you don't want it running `search`/grep either — least privilege by default. |
| **Pin the model** | `model: claude-sonnet-5` for routine test writing; reserve a stronger model for agents that need deep reasoning (e.g. an architecture-review agent). |
| **Scope to an environment** | Add `target: vscode` or `target: github-copilot` if the agent should only run in one surface (e.g. a cloud-only migration agent). |
| **Attach an MCP server** | Add an `mcp-servers` block scoped to just this agent (e.g. giving only the test-writer agent access to a test-coverage reporting MCP tool), instead of exposing it repo-wide. |
| **Control auto-delegation** | Set `disable-model-invocation: true` if the agent should only run when a person explicitly selects it, never via automatic routing. |
| **Chain agents** | Reference a second agent's responsibility in the prompt body ("if the class needs a schema migration, tell the user to switch to the `db-migration` agent") — Copilot doesn't chain agents automatically, so make the handoff explicit. |

### Directory Structure Recommendation (repo-wide)

```
your-repo/
└── .github/
    ├── copilot-instructions.md      # always-on facts: stack, conventions, error handling
    ├── instructions/                # topic-specific guidance files (*.instructions.md)
    │   ├── java-style.instructions.md
    │   └── testing.instructions.md
    └── agents/                      # specialist personas (*.agent.md)
        ├── test-writer.agent.md
        ├── security-auditor.agent.md
        ├── db-migration.agent.md
        └── api-architect.agent.md
```

User-level agents (apply across all your own projects, not committed to the repo) live at:
- **Visual Studio**: `%USERPROFILE%\.github\agents`
- **VS Code / JetBrains**: check your IDE's Copilot settings for the equivalent personal agents folder — as of 2026 this is still evolving per-IDE, so confirm in-app under "Configure Agents."

> **Naming rule**: filenames may only use `. - _ a-z A-Z 0-9`, must end in `.agent.md`, and should be lowercase-hyphenated (`security-auditor.agent.md`, not `SecurityAuditor.agent.md`).

## A.4 Best Practices

- **Lead the `description` with the trigger phrases.** Copilot uses this field to decide when to auto-delegate — write it the way you'd explain the agent's job to a teammate in one sentence, then list 2-3 phrases that should route to it.
- **Constrain tools to the minimum the role needs.** A read-only auditing agent should never have `edit` or `execute`.
- **Keep the prompt focused.** The practical limit is generous (tens of thousands of characters) but concise, concrete instructions consistently outperform long, vague ones — write rules, not essays.
- **State what NOT to do, not just what to do.** "Never modify production code" is more useful to a test-writer agent than another paragraph about what good tests look like.
- **Version-control agent files like code.** They're reviewed in PRs, they drift, and teams that treat `.agent.md` as throwaway config end up with five conflicting "code reviewer" agents within a quarter.
- **Test via explicit selection before trusting auto-delegation.** Confirm the agent behaves correctly when picked manually from the dropdown before relying on Copilot to route to it on its own.

## A.5 Using Your Custom Agent in IntelliJ (Step by Step)

1. **Install/update the GitHub Copilot plugin** from the JetBrains Marketplace (Settings → Plugins → Marketplace → "GitHub Copilot"). Custom agents require a reasonably recent plugin build (1.6.x+ as of mid-2026) — update if you're on an older version.
2. **Sign in** to GitHub Copilot (Business or Enterprise plans get custom agents; confirm your plan includes them).
3. Commit your `.github/agents/*.agent.md` files to the repo (or place them in the user-level agents folder for personal, cross-project agents) and **push/merge to the default branch** — cloud-facing agent discovery reads from the default branch.
4. Open **Copilot Chat** in IntelliJ, click the **Ask/Edit/Agent** mode dropdown.
5. Your custom agent should appear by name in that dropdown. Select it, then chat normally — it now runs under that agent's system prompt and tool restrictions.
6. If it doesn't appear: open **"Configure Agents…"** in the Copilot panel to confirm IntelliJ has indexed the file, and check **Settings → Tools → GitHub Copilot** for any agent-specific toggle.

> **Known-issue note (2026)**: several JetBrains-plugin releases have shipped with bugs where custom agents are correctly detected in settings but don't show up in the chat dropdown (tracked in Microsoft's `copilot-intellij-feedback` repo). If your agent file is confirmed present and valid but missing from the dropdown, this is a plugin bug, not a config error — update the plugin to the latest patch or restart the IDE, and check open issues before assuming your YAML is wrong.

---

# PART B — Claude Code Custom Subagents

## B.1 What it actually is

A subagent is a named, isolated Claude instance with its own system prompt, context window, tool access, and permission mode. The main Claude Code session delegates a scoped task to it; the subagent does the work (reading files, running searches, whatever it needs) and returns **only a summary** to the main conversation — none of its intermediate tool calls pollute your main context. This is the core mechanism for keeping a long Claude Code session from degrading as it fills up with noise.

## B.2 Beginner: Build Your First Sample Subagent

**Goal**: a read-only `code-reviewer` subagent that reports issues by severity without editing anything.

### Step 1 — Create the directory

```
your-project/
└── .claude/
    └── agents/
        └── code-reviewer.md
```

(Use `~/.claude/agents/` instead for a personal subagent available across every project.)

### Step 2 — Write the subagent file

```markdown
---
name: code-reviewer
description: >
  Reviews a recent diff or PR and reports issues by severity
  (blocker/major/minor). Use when asked to "review this PR",
  "check my changes", or "review the diff".
tools: Read, Grep, Glob
---

You are a senior Java/Spring Boot code reviewer.

For the given diff or file set:
1. Classify every finding as BLOCKER, MAJOR, or MINOR.
2. Check for: missing null checks, uncaught exceptions swallowed silently,
   N+1 query patterns, missing @Transactional boundaries, hardcoded secrets,
   and missing/weak test coverage for the changed lines.
3. Do not suggest style-only nitpicks unless nothing else is found.
4. Output a short table: file, line, severity, issue, suggested fix.

You have read-only access. Never propose direct edits — only report findings.
```

### Step 3 — Test it

From the Claude Code CLI in your project:
```
/agents          " lists available subagents, confirms it's registered
```
Then simply ask in a normal session: *"Review my recent changes"* — Claude should recognize the description match and delegate automatically, or you can invoke it explicitly by name.

## B.3 Enhance It (Beginner → Pro)

| Enhancement | How |
|---|---|
| **Tighten tool access** | `tools: Read, Grep` only, if you want to guarantee it can never run `Bash` even accidentally. |
| **Pin a specific model per subagent** | Many public subagent collections set a `model` field so lightweight subagents (doc lookups, log summarization) route to a cheaper/faster model while complex ones (architecture review) use a stronger one. |
| **Make it project-aware** | Add specific knowledge about your stack in the prompt body — e.g. "our repo uses Hikari connection pools; flag any `DataSource` bean not going through `HikariConfig`." This is exactly the kind of narrow expertise that would be noise in your main agent's instructions. |
| **Parallelize related subagents** | You can run `style-checker`, `security-scanner`, and `test-coverage` subagents simultaneously during a single review pass instead of sequentially — each isolated, each returning a summary. |
| **Add explicit stop rules** | State clearly what the subagent must never do (e.g. "never run `git push`", "never modify files outside `src/test`"). |
| **Watch your token budget** | Subagent-heavy workflows can consume roughly **7x** the tokens of a single-thread session, since each subagent spins up its own context. Use subagents for genuinely bounded, noisy work (deep file exploration, log scanning) — not for every trivial question. |

### Directory Structure Recommendation

```
your-project/
└── .claude/
    ├── agents/                      # project-scoped subagents (markdown)
    │   ├── code-reviewer.md
    │   ├── test-gap-analyzer.md
    │   ├── migration-risk-scanner.md
    │   └── log-summarizer.md
    ├── commands/                    # optional: slash-command shortcuts
    └── settings.json                # tool permissions, MAX_SUBAGENTS overrides, etc.

~/.claude/
└── agents/                          # user-level subagents, available in every project
    └── personal-doc-writer.md
```

Environment overrides worth knowing:
- `CLAUDE_CODE_MAX_SUBAGENTS_PER_SESSION` — budget resets on `/clear`.
- `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` — cap on simultaneous subagents.

## B.4 Best Practices

- **Route by task, not by ambition.** Use subagents for bounded work — code review, test-gap analysis, migration-risk scanning, log summarization, a fresh read of a PR. Don't build one broad "developer-agent" that tries to plan, edit, test, and manage the project — that's just a second, more confusing general assistant.
- **Start narrow, project-level, read-only.** Create one focused helper, keep it read-only, test it on a small task, and only widen its tool access once you trust its judgment.
- **Give the subagent everything it needs in the prompt string.** A subagent's context starts fresh — it has no visibility into your main conversation except what you explicitly pass it (file paths, error messages, decisions already made).
- **Don't reach for a subagent for coupled or ambiguous work.** If the task needs constant back-and-forth judgment calls that depend on the main conversation's evolving context, keep it in the main session.
- **Prefer built-ins before building custom.** Claude Code already ships built-in subagents (`Explore`, `Plan`, and a general-purpose worker) — get comfortable with those before writing your own from scratch.

## B.5 Using Claude Code (and Your Subagents) in IntelliJ

Claude Code doesn't have a native IntelliJ chat-panel plugin the way Copilot does — it's used via its CLI, which works fine from IntelliJ's integrated terminal:

1. Install Claude Code (`npm install -g @anthropic-ai/claude-code` or the platform installer — check `docs.claude.com/en/docs/claude-code/overview` for current install steps, since these change).
2. Open IntelliJ's built-in **Terminal** panel (Alt+F12 / View → Tool Windows → Terminal) at your project root.
3. Run `claude` to start a session — it automatically discovers `.claude/agents/*.md` in the open project directory.
4. Ask Claude Code to review, refactor, or investigate as normal; it will delegate to your project's subagents automatically when the task matches a `description`, or you can name one explicitly ("use the code-reviewer subagent on my last commit").
5. Since Claude Code edits files directly on disk, IntelliJ's file-watcher will pick up changes live — keep the relevant files open to see diffs reflected as Claude Code works.

---

# PART C — A Generic LLM Agent, Built From Scratch (Framework-Agnostic)

This is for when neither Copilot's nor Claude Code's built-in agent systems fit — you're building an agent as part of *your own* application or service.

## C.1 The Core Loop (Beginner)

Every agent, regardless of framework, is fundamentally: **system prompt + tool schemas + a loop that calls the model, executes any requested tool, feeds the result back, and repeats until the model returns a final answer.** This is the "ReAct" (Reason + Act) pattern.

### Sample beginner agent (Python, Anthropic API, one tool)

```python
import anthropic

client = anthropic.Anthropic()

def get_order_status(order_id: str) -> str:
    # stub — replace with a real DB/service call
    return f"Order {order_id} is currently PROCESSING."

TOOLS = [
    {
        "name": "get_order_status",
        "description": "Look up the current status of an order by its ID.",
        "input_schema": {
            "type": "object",
            "properties": {"order_id": {"type": "string"}},
            "required": ["order_id"],
        },
    }
]

def run_agent(user_message: str):
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-5",
            max_tokens=1024,
            system="You are a support assistant. Use tools to look up real order data; never guess a status.",
            tools=TOOLS,
            messages=messages,
        )

        if response.stop_reason != "tool_use":
            # final answer — return the text content
            return "".join(b.text for b in response.content if b.type == "text")

        # model wants to call a tool
        messages.append({"role": "assistant", "content": response.content})
        tool_results = []
        for block in response.content:
            if block.type == "tool_use" and block.name == "get_order_status":
                result = get_order_status(block.input["order_id"])
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })
        messages.append({"role": "user", "content": tool_results})

print(run_agent("What's the status of order 12345?"))
```

This is intentionally minimal — no memory, no retries, no logging. That's the beginner baseline; everything below is how you take it to production.

## C.2 Directory Structure Recommendation (Real Project)

```
my-agent/
├── agent/
│   ├── loop.py               # the core ReAct loop / orchestration
│   ├── prompts/
│   │   ├── system_prompt.md  # keep prompts OUT of Python strings
│   │   └── planning_prompt.md
│   └── config.yaml           # model names, token limits, timeouts
├── tools/
│   ├── __init__.py           # tool registry (name -> function + schema)
│   ├── order_lookup.py
│   ├── inventory_check.py
│   └── base.py                # shared Tool interface/contract
├── memory/
│   └── conversation_store.py  # session state, if any
├── evals/
│   ├── test_cases.jsonl       # input/expected-output pairs
│   └── run_evals.py           # scores agent responses against test_cases
├── observability/
│   └── tracing.py             # structured logging of every tool call, token count
├── tests/
│   └── test_tools.py          # unit tests for tools themselves, independent of the LLM
└── main.py
```

**Why this shape**: prompts live as versioned files, not inline strings (you'll iterate on wording constantly — treat it like config, review it in PRs). Tools are isolated and independently testable without invoking the model at all. Evals are a first-class folder, not an afterthought — see below.

## C.3 Enhancing to Pro Level

| Beginner gap | Pro-level fix |
|---|---|
| No error handling on tool calls | Wrap every tool execution in try/except; return a structured error to the model ("tool failed: connection timeout") instead of crashing the loop — a well-instructed model can often recover or ask the user for guidance. |
| No limit on loop iterations | Cap the ReAct loop (e.g. max 8 tool-call rounds) and fail gracefully with a clear message rather than looping indefinitely on a confused model. |
| No structured output validation | If you need reliable structured output (JSON for downstream systems), constrain the final response format explicitly and validate it against a schema before trusting it — reject and re-prompt on validation failure rather than passing bad data downstream. |
| No observability | Log every tool call, its input/output, latency, and token usage per turn. Without this you cannot debug a bad agent run or measure cost. |
| No evals | Build a small, versioned set of realistic input/expected-behavior pairs and re-run them whenever you change the prompt or tool set — this is the only reliable way to know a prompt "improvement" didn't regress something else. |
| Single fixed model for everything | Add model routing: cheap/fast model for simple classification or extraction sub-steps, escalate to a stronger model only for the steps that need real reasoning (see Part D). |
| Tool descriptions are vague | Tool descriptions are part of the prompt the model reasons over — write them as carefully as you'd write documentation for a teammate; vague tool descriptions are one of the most common causes of an agent calling the wrong tool or the right tool with wrong arguments. |
| No human-in-the-loop for risky actions | For anything destructive or hard to reverse (deleting data, sending real emails, spending money), require explicit confirmation before executing — don't let the agent's autonomy outrun your risk tolerance. |

## C.4 Cross-Cutting Best Practices (All Three Agent Types)

- **Single responsibility per agent.** A "does everything" agent is just a worse general assistant with extra confusion — the value of a custom/subagent comes from narrow, well-defined scope.
- **Tool minimalism (least privilege).** Every tool you grant is a capability you have to trust the model to use correctly and safely. Grant only what the role needs.
- **Prompts are code — version them, review them, test them.** Treat prompt changes with the same rigor as logic changes, because for an agent, they largely are the logic.
- **State boundaries explicitly, not just capabilities.** "What this agent should never do" prevents more real incidents than another paragraph of "what it should do."
- **Test before trusting auto-delegation/automation.** Whether it's a Copilot agent's `description`-based routing or a subagent's implicit trigger, verify manually first.
- **Design for observability from day one.** You cannot improve — or safely debug — what you can't see happening.

---

# PART D — The Pro-Level, Token-Optimized Agent (Same Accuracy, Lower Cost)

This section assumes the Part C architecture and layers in token-efficiency techniques that **do not** trade away output quality — they remove genuine waste, not useful context.

## D.1 The Core Techniques

1. **Prompt caching for anything stable.** Your system prompt, tool schemas, and any long fixed reference material (a style guide, a schema dump) rarely change turn to turn. Anthropic's prompt caching lets you mark a stable prefix so repeated requests reuse the already-processed representation instead of paying full price on every call. As of the current API, caching comes in two modes:
   - **Automatic** — set a single top-level `cache_control` field and the API applies the cache breakpoint to the last cacheable block for you.
   - **Explicit** — place `cache_control: {"type": "ephemeral"}` on specific content blocks (e.g. the end of your system prompt) for fine-grained control over exactly what's cached.
   Cache entries default to a 5-minute lifetime; a 1-hour extended cache is available at extra cost via `"ttl": "1h"` for workloads with longer gaps between reuses. **The cache is prefix-based and exact** — any change earlier in the request (including things like `tool_choice` or the presence of an image) invalidates it, so keep genuinely-stable content first and dynamic content (the actual user turn) last.

2. **Model routing / tiering.** Not every step in an agent's workflow needs your strongest model. A common pattern:
   - Cheap/fast model for triage, classification, extraction, "is this even relevant" filtering.
   - Strong model only for the steps that need real judgment (final synthesis, ambiguous reasoning, code review verdicts).
   This alone often cuts total cost dramatically since most agent workflows are triage-heavy and judgment-light.

3. **Retrieval over context-stuffing.** Don't paste an entire file, log, or document into context "just in case." Fetch or grep only the relevant section. This is exactly why Claude Code subagents exist — isolate the noisy exploration, return only the summary.

4. **Concise system prompts, bullet-first.** Verbose prose system prompts cost tokens on every single call unless cached. Prefer scannable rules over paragraphs — and if caching, remember length still matters for the *first* call that creates the cache entry.

5. **Constrained output format.** If downstream systems just need a status and a short reason, don't let the model produce a five-paragraph explanation by default — specify the exact output shape you need (a small JSON object, a one-line verdict) so output tokens (which you pay for and which take time to generate) aren't wasted on unrequested elaboration.

6. **Avoid re-describing the same tools on every call inconsistently.** Keep tool definitions byte-for-byte stable between calls in the same session so they stay inside the cached prefix — a tool schema hand-generated fresh each turn (even if semantically identical) breaks the exact-prefix-match cache.

7. **Batch when latency isn't critical.** For non-interactive workloads (nightly log summarization, bulk PR triage), batch processing APIs are typically priced lower than synchronous calls — use them when you don't need an answer in the next few seconds.

8. **Summarize before re-injecting long tool results.** If a tool call returns a huge payload (a full log dump, a large file), have your own code summarize/truncate it before it goes back into the model's context, rather than feeding the raw payload back on every subsequent turn of a multi-turn tool loop.

## D.2 Sample: A Token-Optimized Code-Review Agent

This extends the Part C skeleton with caching, tiering, and constrained output — same review quality, meaningfully fewer tokens per PR.

```python
import anthropic, json

client = anthropic.Anthropic()

SYSTEM_PROMPT = """You are a senior code reviewer for a Java/Spring Boot monorepo.
Rules:
- Flag only BLOCKER and MAJOR issues by default; MINOR only if nothing else found.
- Categories: null-safety, transaction boundaries, N+1 queries, secrets, missing tests.
- Output ONLY a JSON array: [{"file","line","severity","issue"}]. No prose, no markdown fencing.
"""  # kept short and stable -> ideal cache candidate

def triage_files(changed_files: list[str]) -> list[str]:
    """Cheap/fast model: filter out files that obviously don't need deep review
    (pure formatting diffs, generated code, test-only additions with no logic)."""
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",   # cheap tier for triage
        max_tokens=256,
        system="Return ONLY a JSON array of filenames worth a deep code review; drop pure formatting/generated/test-only diffs.",
        messages=[{"role": "user", "content": json.dumps(changed_files)}],
    )
    text = "".join(b.text for b in response.content if b.type == "text")
    return json.loads(text)

def deep_review(file_diff: str) -> list[dict]:
    """Strong model, cached system prompt, constrained JSON output."""
    response = client.messages.create(
        model="claude-sonnet-5",              # strong tier for real judgment
        max_tokens=512,
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"},   # cache the stable instructions
            }
        ],
        messages=[{"role": "user", "content": file_diff}],  # only the dynamic part is uncached
    )
    text = "".join(b.text for b in response.content if b.type == "text")
    return json.loads(text)  # validate/raise if malformed rather than trusting blindly

def review_pr(changed_files: dict[str, str]) -> list[dict]:
    worth_reviewing = triage_files(list(changed_files.keys()))
    findings = []
    for f in worth_reviewing:
        findings.extend(deep_review(changed_files[f]))
    return findings
```

**What changed vs. the naive version, and why it doesn't cost accuracy:**
- Files that are obviously safe (formatting-only, generated code) never reach the expensive model at all — a cheap classification step, not a quality cut.
- The system prompt is marked for caching, so every subsequent file in the same PR review reuses it instead of re-paying for the same ~150 tokens of instructions each time.
- Output is constrained to structured JSON — no wasted generation tokens on conversational filler, and the strict schema means a downstream dashboard can consume it directly without another parsing pass.
- The review logic itself (the actual judgment) still runs on the strong model with the full instructions — nothing about the *review quality* was reduced, only the waste around it.

## D.3 Directory Structure for the Pro Agent

```
pro-agent/
├── agent/
│   ├── triage.py            # cheap-tier classification step
│   ├── review.py            # strong-tier judgment step, cached system prompt
│   └── prompts/
│       └── system_prompt.md # short, stable, cache-friendly
├── config/
│   └── model_tiers.yaml     # which model handles which step; easy to re-tune
├── observability/
│   └── token_metrics.py     # logs cache_read_input_tokens vs cache_creation_input_tokens,
│                             # per-step cost, so you can see caching actually working
└── evals/
    └── regression_suite.jsonl  # re-run after ANY prompt/model-tier change
```

## D.4 Measuring That You Actually Improved Something

Every response from the API returns usage details — track these, don't just assume caching/tiering worked:

```json
{
  "usage": {
    "input_tokens": 2048,
    "cache_read_input_tokens": 1800,
    "cache_creation_input_tokens": 248,
    "output_tokens": 503
  }
}
```

A healthy pattern over a session looks like: `cache_creation_input_tokens` is non-zero on the *first* call (cache being built), then subsequent calls in the same window show a high `cache_read_input_tokens` relative to `input_tokens` — that's your evidence caching is actually firing, not just configured. If `cache_read_input_tokens` stays at zero across repeated calls with an identical prefix, something in your request (a reformatted history, a changed tool list, an inconsistent prefix) is breaking the cache — investigate before assuming the feature "doesn't work."

**Before shipping any token-optimization change**, re-run your eval suite (§C.3) and confirm pass rate is unchanged. Token optimization that quietly degrades accuracy isn't optimization — it's a regression wearing a lower invoice.

---

# Quick Reference: Where Everything Lives

| Artifact | Path |
|---|---|
| GitHub Copilot custom agent (repo) | `.github/agents/*.agent.md` |
| GitHub Copilot custom instructions (always-on) | `.github/copilot-instructions.md` |
| GitHub Copilot topic instructions | `.github/instructions/*.instructions.md` |
| Claude Code subagent (project) | `.claude/agents/*.md` |
| Claude Code subagent (personal, all projects) | `~/.claude/agents/*.md` |
| Claude Code settings/permissions | `.claude/settings.json` |
| Generic agent prompts (recommended) | `agent/prompts/*.md`, versioned like code |
| Generic agent evals (recommended) | `evals/*.jsonl` + a runner script |

---

## Sources Consulted (verified current as of July 2026)

- [VS Code — Custom agents](https://code.visualstudio.com/docs/agent-customization/custom-agents)
- [GitHub Docs — Custom agents configuration](https://docs.github.com/en/copilot/reference/custom-agents-configuration)
- [GitHub Docs — Creating custom agents for Copilot cloud agent](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/create-custom-agents)
- [Microsoft Learn — Use custom agents in GitHub Copilot (Visual Studio)](https://learn.microsoft.com/en-us/visualstudio/ide/copilot-specialized-agents?view=visualstudio)
- [GitHub Copilot IntelliJ feedback repo — custom agent GA & known dropdown issues](https://github.com/microsoft/copilot-intellij-feedback)
- [Claude Code — Subagents in the SDK](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Claude Code Subagents: A Practical 2026 Guide — Nimbalyst](https://nimbalyst.com/blog/claude-code-subagents-guide/)
- [Anthropic — Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Anthropic Product Docs Map](https://docs.claude.com/en/docs/claude-code/overview) (always check for the latest install/config specifics — these evolve quickly)

> Note: several details here (plugin version numbers, exact IntelliJ menu paths, GA dates) are current as of mid-2026 but move fast — re-check the linked official docs before treating any specific version number as a long-term fact.
