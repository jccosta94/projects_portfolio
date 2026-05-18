# MCP Integration & Pattern C

The canonical Hugo skill template. How specialist capabilities are wrapped so the agent's role is reduced to summarisation, and the failure modes of cheap models become non-issues.

## The problem

Hugo runs on `gpt-5.4-mini` via Codex CLI. The model is fast and cheap enough to make per-customer ChatGPT Plus subscriptions viable. But it has a specific limitation: **it does not reliably bind soft procedural framing.**

In practice this means: when a SKILL.md prompts the model with *"first call X, then call Y with X's output, then summarise"* — if step 1 has any plausible alternative (ask the customer, fabricate from context, look at session memory), the model takes the alternative. The procedural contract is treated as a suggestion.

This failure mode broke three separate skill implementations during the OpenClaw multi-agent era (Issues #83, #81, #86 — full narrative in [`../case-study/architecture-evolution.md`](../case-study/architecture-evolution.md)). Each was the same shape: a multi-step delegation flow where the model resolved soft preconditions by improvising rather than executing.

**Pattern C is the architectural fix.**

## What it is

For any specialist capability, all deterministic mechanical work goes in a **single MCP tool call**. Hugo's role is to:

1. Invoke the tool once
2. Read the tool's return value
3. Synthesise the customer-facing summary from that return value

Hugo does not perform multi-step procedural work inline. Hugo does not "remember" what the tool did in a previous turn. Hugo does not infer state from BCP contents or session memory.

The tool either returned a result or it didn't. Hugo synthesises **from** the result.

## Why it works

The failure mode requires soft preconditions to fail on. Pattern C eliminates them.

There's no "ask the user first" escape valve because the tool doesn't have a question-asking step. There's no "fabricate plausibly" path because the synthesis source is a concrete return value, not a probability distribution over plausible inputs. There's no "check session memory" shortcut because the agent's role is explicitly to summarise the latest tool return, not to infer current state from history.

The model still chooses *whether* to call the tool, *which* tool to call, and *how* to phrase the customer summary. But the deterministic work — extract URLs from a brand context profile, fetch a page, write a markdown artifact, query an API — is happening in Python, not in the model.

## SKILL.md — the six canonical sections

Every Hugo SKILL.md follows the same six-section structure.

### 1. Signature triggers

A mechanical phrase list, not intent-based. The `description` field is operationally load-bearing on Telegram: it drives auto-skill-selection against a pool of 87+ Hermes built-ins. Without a strong-keyword description, Hermes picks a built-in over the Hugo skill. (We learned this when Hugo picked the `xurl` built-in over `post-ad-hoc` on a "post to X" trigger — Issue #97.)

### 2. Numbered tiers

A 3-tier structure with explicit conditions:

- **Tier 1** — proceed autonomously (internal work, no spend, no public action)
- **Tier 2** — proceed with notice (bounded spend or reach, code-enforced caps)
- **Tier 3** — require explicit APPROVE before any action (public posts, ad launches, landing pages, outbound email)

### 3. GOOD / BAD examples

Concrete, side-by-side. The bad examples are real failures from previous skill iterations; the good examples are the corrected patterns.

### 4. Counterfactual

The hardest gate. Verbatim form:

> *"if no `<tool_name>` result in this turn's tool history, `<step>` has not run; do NOT compose `<next step>`."*

Without this, Mini will synthesise plausibly. With it, the skill refuses to fabricate — because the SKILL.md tells it explicitly that the precondition is the tool return, not anything else.

### 5. Forbidden-tokens guardrail

A mechanical list of phrases Hugo must never emit in the customer channel: model names, internal paths, tool-progress text, framework names. The customer talks to Hugo, not to a stack.

### 6. State-readback rule

> *"Before answering any 'did you finish?' or 'what did you find?' follow-up, re-invoke the same MCP tool with `mode: "readback"` to retrieve the on-disk artifact; do not replay from session memory and do not read the file directly."*

This closes the state-fabrication failure class (Mini answering "yes, it's done" from session memory without verifying).

## MCP server scaffolding

Every Hugo MCP server lives at `~/.hermes/mcp-servers/<domain>/server.py` with a `run.sh` wrapper script.

### PEP 723 inline dependencies

Each server declares its dependencies in a `# /// script` block at the top of `server.py`, which `uv run --script` reads natively. No `pyproject.toml`, no `setup.py`.

```python
# /// script
# requires-python = ">=3.11"
# dependencies = [
#     "mcp>=1.0.0",
#     "httpx>=0.27.0",
#     "beautifulsoup4>=4.12.0",
# ]
# ///
```

### The `run.sh` shim

Hermes registers MCP servers via `hermes mcp add <name> --command bash --args <path/run.sh>`. The shim exists because Hermes's `--script` argparse flag collides with `uv run --script`. `run.sh` wraps the invocation:

```bash
#!/usr/bin/env bash
exec uv run --script "$(dirname "$0")/server.py" "$@"
```

### Area-scoped server naming

Servers are named for their functional area, not for the single skill they expose at creation time. A server named `planning/` holds `compose_plan` today and will hold `revise_plan` tomorrow. Don't create one server per skill. Naming convention: noun-form domain area (`planning/`, `audit/`, `reporting/`, `content-plan/`).

## State-readback via `mode: "readback"`

When the customer asks again about a previously-generated artifact ("did you finish the audit?"), Hugo re-invokes the same MCP tool that produced it, passing `mode: "readback"` alongside the `tenant_id`. The tool reads the artifact from disk and returns its contents. Hugo synthesises the customer-facing summary from that return value.

Hugo does NOT:

- Read the artifact file directly (Hugo doesn't hold the absolute artifact path; it's internal to the tool)
- Replay from session memory ("I think I said it was done")
- Make up the summary

A `readback` invocation never writes a new artifact and never bumps a version number. It's a pure retrieval call.

This pattern closes the state-fabrication failure class. Every state query stays grounded in a live tool call.

## Hugo-scoped env var discipline

Every external API integration in Hugo uses Hugo-scoped env var names — never the vendor SDK's default pickup name.

For example: `GOOGLE_AI_PRO_API_KEY` (not `GEMINI_API_KEY` or `GOOGLE_API_KEY`), `UPLOAD_POST_API_KEY` (not a generic `API_KEY`).

The rationale: a host machine or shared Hermes container may have global vendor env vars set for unrelated purposes. If a Hugo MCP server relied on the SDK's default env-var pickup, it would silently pick up the wrong key and produce authenticated-but-wrong API calls with no error surfaced. The failure mode is invisible: the SDK succeeds at authentication, but the key belongs to the wrong account or project.

The corresponding code-side rule: every SDK call passes the API key via explicit configuration, not by relying on SDK auto-pickup.

```python
# Correct — explicit, Hugo-scoped
genai.configure(api_key=os.environ["GOOGLE_AI_PRO_API_KEY"])

# Wrong — silently picks up GOOGLE_API_KEY from environment
genai.configure()
```

Three places, consistent naming: the `.env` file, the `os.environ[]` call, the install script's secret-collection step. No ambiguity.

## Skill location and allowlist

Hugo skills live at `~/.hermes/skills/hugo/<skill-name>/SKILL.md`. The directory layout (`category == "hugo"` derived from path) is the discriminator — no manifest field required.

The Telegram allowlist is auto-managed by `~/.hermes/scripts/sync-hugo-skill-allowlist.py`. The script maintains `skills.platform_disabled.telegram` in `~/.hermes/config.yaml`: every non-Hugo Hermes built-in is disabled on Telegram; every Hugo skill stays enabled automatically. Idempotent, with `--check` mode for CI-style drift detection.

Why the allowlist matters: on Telegram, Hermes auto-picks skills by description-keyword overlap from the full 87+ built-in pool. Without the allowlist, a customer message like "post this to X" can route to Hermes's `xurl` built-in instead of Hugo's `post-ad-hoc`. The built-in's description is a stronger keyword match unless the Hugo skill is the only candidate.

## Validation count

Pattern C has been validated five times across V1 shipped features.

| Skill | Server | Issue |
|---|---|---|
| `post-ad-hoc` | Upload-Post shell wrapper | #97 |
| `presence-audit` | `~/.hermes/mcp-servers/audit/` | #100 |
| `post-from-content-plan` | `~/.hermes/mcp-servers/content-plan/` | #102 |
| `reporting-lead` | `~/.hermes/mcp-servers/reporting/` | #104 |
| `compose-plan` | `~/.hermes/mcp-servers/planning/` | #83 (Pattern C origin) |

Each one follows the same shape: SKILL.md with the six canonical sections, MCP server with PEP 723 + `run.sh`, agent invokes the tool once and synthesises the customer summary from the return.

## What Pattern C is not good for

- **Capabilities that genuinely need multi-step user interaction.** E.g., a guided onboarding flow where the customer answers a series of questions and each answer shapes the next. Pattern C would force this into one tool call with all the branching logic in Python, which loses the conversational quality.
- **Long-running async work where the customer needs progress updates.** A 30-minute campaign rollout doesn't fit one synchronous tool call. Needs a different pattern (deferred to post-V1).
- **Capabilities where the synthesis itself is the value.** E.g., creative writing tasks where the model's voice matters as much as the structured output. Pattern C reduces synthesis to summarisation; for writing-as-output, the model's contribution should be larger.

For everything else — extract, fetch, compute, write — Pattern C is the canonical shape.
