# Hugo — AI Marketing Employee
**A multi-tenant AI marketing system, in two architectural generations. Designed as a team, re-architected as a single agent with specialist tools.**

This is a case study in **multi-agent system design under cost-tier model capability** — specifically: how a 7-agent marketing-team architecture broke when the cheap model (cheap enough to make per-customer subscriptions viable) couldn't bind soft procedural framing, and what replaced it.

The system runs at [jccosta94.github.io/hugo-website](https://jccosta94.github.io/hugo-website/), with the first customer onboarding in progress (a paint manufacturer pivoting from B2B distribution to consumer DIY). The constraint that triggered the architecture work: each tenant runs on the customer's own ChatGPT Plus subscription for inference. That subscription gates Hugo to `gpt-5.4-mini` — fast, cheap, but capability-constrained. The 7-agent design assumed the model could reliably bind multi-step delegation contracts. It can't.

### What I architected

**v1 — a hierarchical 7-agent team** (Director + Performance Marketer + Web Developer + Brand Designer + Community Manager + Outbound Specialist + Analytics) on top of [Hermes Agent](https://hermes-agent.nousresearch.com/) (then briefly [OpenClaw](https://openclaw.ai/) for multi-agent dispatch primitives). Director is customer-facing on Telegram. Specialists are dispatched by Director, return results, Director synthesizes back to the customer — the same hierarchical org-chart pattern that works for engineering teams (and for Psinest's build pipeline). **Three independent multi-agent delegation failures broke this design before V1 shipped to customers.** All three traced to the same root cause: when the SKILL.md tells `gpt-5.4-mini` to "first call X, then synthesize Y from X's output," and step X has any plausible alternative (ask the customer, fabricate from context, replay from session memory), the model takes the alternative. The procedural contract is treated as a suggestion.

**v2 — a single Hugo agent + specialists-as-MCP-tools** on Hermes Agent (MIT, [Nous Research](https://nousresearch.com)). Specialist capabilities collapsed from separate agents to area-scoped MCP tool servers (`marketing__*`, `creative__*`, `web__*`, `inbox__*`, `outbound__*`, `analytics__*`, `audit__*`, `content_plan__*`). The agent's only job: invoke one tool per turn, synthesize the customer-facing summary from the tool's return value. I called this **Pattern C** and codified it as the canonical Hugo skill template — a 6-section SKILL.md with a counterfactual guardrail, a forbidden-tokens list, and state-readback via `mode: "readback"`. **Pattern C removes the failure surface entirely**: there are no soft preconditions for the model to fail on. The tool either returned a result or it didn't. The agent synthesizes from the result.

### The architectural insight

**Model capability is a topology constraint.** When the model is locked in — because it's what the per-customer subscription tier gives you — it cannot be upgraded as a substitute for fixing the architecture. The failure mode of `gpt-5.4-mini` ("synthesize plausible content from soft preconditions") exists at a specific scale of model capability. The next tier up (`gpt-5-pro`) might bind multi-step delegation reliably, but at a price point that breaks the per-customer subscription model. The model can't change; the topology has to.

Pattern C is the architectural response: collapse the procedural contract into a single tool call. The agent's behaviour reduces from "execute a procedure" to "summarize a tool return." There's nothing for the failure mode to attach to.

**A secondary benefit, not the original motivation but very real once you're running both shapes: v2 is materially cheaper to operate.** v1 burns 5–7 inference calls per customer turn (Director translates the brief → dispatches to specialists → each specialist generates output → Director synthesizes the response back). v2 burns 1–2 calls per turn (the agent picks a tool, the tool does the deterministic work in Python, the agent synthesizes the return). Same per-customer subscription tier, much lower rate-limit consumption, more headroom per pod. **Pattern C didn't just remove the failure surface — it also removed the work the model didn't need to be doing.**

**Two designs, sequential. Same customer surface on top. Same model doing the same things in both — but only one of them produces reliable output, and only one of them does it at a rate the per-customer subscription can sustainably support.**

---

## TL;DR

| | **v1 — multi-agent team** | **v2 — single agent + MCP tools** |
|---|---|---|
| **Solves which problem** | **Coverage** — map 6 marketing specialties to 6 specialist agents | **Capability** — make the cheap model produce reliable customer output |
| **Topology** | 7 agents (Director + 6 specialists) in hierarchical dispatch | 1 agent + 8 MCP tool namespaces |
| **Customer touchpoint** | Director on Telegram, opaque to specialist work | Hugo on Telegram, opaque to MCP tool calls |
| **Inference model** | `gpt-5.4-mini` per agent (per-customer ChatGPT Plus) | `gpt-5.4-mini`, single agent (per-customer ChatGPT Plus) |
| **Where deterministic work happens** | In specialist agents' system prompts (in the model) | In Python (in MCP tool servers) |
| **What the agent does** | Decides what to do, delegates, synthesizes specialist outputs | Decides what to do, picks a tool, synthesizes tool output |
| **Inference calls per customer turn** | 5–7 (brief translation + dispatch + specialists + synthesis) | 1–2 (tool selection + summary of tool return) |
| **Failure mode** | Model fabricates plausible specialist output when delegation contracts have soft preconditions | None — Pattern C eliminates soft preconditions |
| **Cross-customer isolation** | Per-tenant Docker container + per-agent SQLite memory | Per-tenant Docker container + per-tenant SQLite memory |
| **Customer-facing channel** | Telegram (default) · Slack / WhatsApp / Signal / Email (alternates) | Same |
| **Operating state today** | Dropped before V1 ship | Current architecture. 5 skills shipped following Pattern C. |
| **Reversibility** | Available if a future model class binds multi-step delegation reliably — MCP tools can be refactored back into agents | N/A — current architecture |

**The headline insight: same model, same customer surface, completely different internal architecture.** The customer-facing experience never changed. The internal failure surface went from "every multi-step delegation contract is a probabilistic gamble" to "the model summarizes tool returns — that's all."

---

## Why this matters

Most teams building AI agent products in 2026 hit two walls in sequence: **first the topology, then the capability.**

**Wall 1 — Topology.** A multi-agent design feels natural for any product where the work decomposes into specialties (marketing has 6 of these; software development has 7). The temptation is to map specialties to agents 1:1 and let an orchestrator handle dispatch. **This is what Hugo v1 tried.** It can work if the model can bind the multi-step delegation contract. For cheap models — and "cheap" is exactly the tier you need to make per-customer subscription pricing viable — it doesn't bind. The failure mode is silent: the model synthesizes plausible content from soft preconditions, and the customer can't tell whether the work was done or hallucinated.

**Wall 2 — Capability.** Once you accept that the cheap model can't bind multi-step delegation, the obvious move is to upgrade. **But the cheap model isn't a choice — it's a price-point constraint.** Hugo's per-customer subscription model only works at the ChatGPT Plus tier. Upgrading to ChatGPT Pro multiplies that cost per customer. Upgrading to direct API access multiplies it further and removes the per-customer-pays property entirely. The model is locked in. **This is where Pattern C comes in:** if the model can't bind the contract, remove the contract.

The lesson is **the sequence.** Build v1 first, even knowing it might not survive — that's what proves the multi-step contract is actually the failure surface (not "we wrote a bad prompt," not "we picked the wrong specialist split"). Once the diagnosis is clear, the topology change is straightforward and the customer experience is preserved. **Doing them in the wrong order — guessing the diagnosis without v1 evidence, or sticking with v1 once the failures pile up — leads to either underbuilt systems or stranded effort.**

---

## v1 — the 7-agent team

The original Hugo design (April 2026) was a hierarchical agent team modelled after a marketing agency's org chart. Director is the customer interface; six specialists handle domain work; Director synthesizes specialist output into customer-facing replies.

### The team

```
                      Telegram (per-tenant bot)
                                │
                                ▼
                       ┌──────────────────┐
                       │   Director 🎯    │   ← customer-facing
                       │  (gpt-5.4-mini)  │
                       └────────┬─────────┘
                                │ dispatches to specialists
            ┌──────────┬────────┼────────┬───────────┬──────────┐
            ▼          ▼        ▼        ▼           ▼          ▼
       ┌────────┐ ┌────────┐ ┌──────┐ ┌──────┐ ┌──────────┐ ┌──────────┐
       │ Perf 📈│ │ Web 💻 │ │Brand │ │Inbox │ │ Outbound │ │ Analytics│
       │ Marketer│ │ Dev    │ │ 🎨   │ │ 💬   │ │   📧     │ │   📊     │
       └────────┘ └────────┘ └──────┘ └──────┘ └──────────┘ └──────────┘
                  all specialists: gpt-5.4-mini · terminal (no further dispatch)
```

### Agent roster

| Agent | Role | Domain | Returns to Director |
|---|---|---|---|
| **Director** 🎯 | Customer-facing orchestrator | Channel I/O · brief translation · synthesis | Customer-facing message |
| **Performance Marketer** 📈 | Google + Meta Ads | Ad strategy, budget allocation, A/B testing | Structured campaign brief, ad copy drafts |
| **Web Developer** 💻 | Landing pages + Google Business Profile | HTML / Cloudflare Pages | Deployed page URL + summary |
| **Brand Designer** 🎨 | Image + video generation | Veo 3 + Imagen + Nano Banana | Asset URLs + descriptions |
| **Community Manager** 💬 | Inbound DM / comment triage | Instagram / Facebook (V1: IG-only) | Reply suggestions + flagged threads |
| **Outbound Specialist** 📧 | Cold email + SEO comments | SendGrid + web research | Outreach drafts + send queue |
| **Analytics** 📊 | GA4 + Meta Ads + Stripe pulls | Python via Codex CLI | CAC / ROAS / structured findings |

All specialists are terminal — they don't dispatch further. This keeps the org chart shallow (2 levels deep: Director → Specialist) and matches how real marketing agencies operate.

### The customer flow (designed)

A typical campaign workflow under v1:

```
1. Customer (owner) on Telegram:  "we're doing a 20% black friday sale next week, get the word out"
2. Director translates to a structured campaign brief
3. Director dispatches in parallel:
   ├─ Performance Marketer  → drafts Google + Meta ad copy
   ├─ Web Developer         → builds landing page
   ├─ Brand Designer        → generates campaign creative
   └─ Analytics             → forecasts CAC / ROAS from prior campaigns
4. Specialists return outputs to Director
5. Director synthesizes: "Campaign ready. Landing page [URL] · ad previews [images] 
   · forecast [budget] · APPROVE to launch"
6. Customer replies APPROVE
7. Director triggers Performance Marketer to launch ads, Web Dev to push the page live, 
   Brand Designer to schedule creative via Upload-Post
```

The design assumed Director could reliably bind a multi-step delegation contract: *"first dispatch specialists, wait for returns, then synthesize."* The model couldn't.

### How v1 broke

Three failures, all in early May 2026, all under `gpt-5.4-mini`, all the same shape:

**Issue #83 — audit delegation.** Director was supposed to invoke Performance Marketer to run an account audit. Instead, Director made up audit findings. The model decided synthesizing plausible audit content was an easier path than executing the delegation contract.

**Issue #81 — Brand Designer topic confab.** Director was supposed to convene a multi-step Brand Designer skill to draft a topic. Instead, it asked the customer for input and fabricated the topic from that. The model took the "ask the customer" escape valve instead of running the prescribed flow.

**Issue #86 — Director → Brand Designer post drafting.** Same pattern: a delegation contract with soft preconditions, the model resolved them by improvising rather than dispatching.

Three failures, one diagnosis: **`gpt-5.4-mini` doesn't bind soft procedural framing.** When you tell the model "first call X, then synthesize Y from X's output, do not skip step 1," and step 1 has any plausible alternative (ask the user, fabricate from context, look at session memory), the model takes the alternative. The procedural contract is treated as a suggestion.

Two responses:

- **Fix the model.** Wait for a more capable tier — `gpt-5-pro` via Codex CLI was a plan B option. But that breaks the per-customer ChatGPT Plus subscription model that makes Hugo's economics work. Indefinite wait, indefinite cost.
- **Fix the topology.** Collapse the multi-step delegation into a single tool call. Remove the soft preconditions. Reduce the agent's role to summarizing tool returns.

Fixing the topology was the shorter path, didn't require a price-point change, **and as a side effect dramatically reduced inference work per customer turn** — 5–7 calls collapsed to 1–2, with the deterministic work moved into Python (where it costs nothing). **That became v2.**

---

## v2 — single Hugo agent + MCP tools (Pattern C)

The collapse: 7 agents → 1 agent + 8 MCP tool namespaces. Customer-facing experience unchanged. Internal architecture completely different.

### Architecture

```
                       Customer · Telegram (or Slack / WhatsApp / Signal / Email)
                                  |
                                  |  messages · voice · files · APPROVE replies
                                  v

                       +-----------------------------------+
                       | CHANNEL ADAPTER (Hermes)          |
                       |   outbound bot · per-pod          |
                       |   APPROVE recognizer              |
                       +----------------+------------------+
                                        |
                                        v
                       +-----------------------------------+
                       | HUGO AGENT                        |   <-- customer-facing
                       |   gpt-5.4-mini                    |
                       |   single-agent topology           |
                       |   skill picker                    |
                       |   tool dispatcher                 |
                       |   summarizer (Pattern C)          |
                       +---+--------------------------+----+
                           |                          |
            shell-out per turn         one MCP tool call per turn
                           v                          v
                +---------------------+   +-----------------------------+
                | CODEX CLI BRAIN     |   | MCP TOOL REGISTRY           |
                |   model:            |   |   Pattern C dispatch        |
                |   gpt-5.4-mini      |   |   tool returns deterministic|
                |   per-pod ChatGPT   |   |   result; agent synthesizes |
                |   Plus subscription |   |   customer reply from return|
                |   (customer's)      |   |                             |
                +---------------------+   +---------------+-------------+
                                                          |
                                       eight area-scoped tool servers
                                                          |
+------------------+------------------+------------------+------------------+
| marketing__      | creative__       | web__            | inbox__          |
|   Google Ads     |   Veo 3          |   Cloudflare     |   Upload-Post    |
|   Meta Ads       |   Imagen         |   Pages          |   inbox API      |
|   manager OAuth  |   Nano Banana    |   landing pages  |   IG-only V1     |
|   customer bills |   Google AI Pro  |   Google Biz     |   5-min polling  |
|   direct         |   per customer   |   Profile        |                  |
+------------------+------------------+------------------+------------------+
| outbound__       | analytics__      | audit__          | content_plan__   |
|   SendGrid       |   GA4 read-only  |   URL fetches    |   scheduled      |
|   cold email     |   Stripe r-only  |   + Beautiful    |   posts from     |
|   SEO comment    |   Codex Python   |   Soup parsing   |   BCP plan       |
|   placement      |   live CAC/ROAS  |   BCP URL extract|   via Upload-    |
|                  |                  |   presence audit |   Post · idemp   |
+------------------+------------------+------------------+------------------+

Each MCP server: Python · PEP 723 inline deps · uv run --script · run.sh shim
                 area-scoped naming (multiple skills per server)
                 mode: "readback" supported for state retrieval
                 Hugo-scoped env vars (never SDK auto-pickup)

-------------- inside the pod · governance + state --------------

+----------------------------------+  +----------------------------------+
| HITL GATE (code-enforced)        |  | POD STATE + MEMORY               |
+----------------------------------+  +----------------------------------+
|   Tier 1  auto      internal     |  |   ~/.hermes/sessions.db          |
|   Tier 2  capped    bounded reach|  |     (SQLite, per-pod)            |
|   Tier 3  HITL      APPROVE      |  |                                  |
|                                  |  |   tenants/<id>/context/          |
|   APPROVE recognizer:            |  |     BCP · plans · audits         |
|     phrase list match            |  |     reports · posts · goals      |
|                                  |  |                                  |
|   Forbidden tokens guardrail:    |  |   tenants/<id>/config/.env       |
|     never emit model / path /    |  |     chmod 600 · never in git     |
|     framework names              |  |                                  |
+----------------------------------+  +----------------------------------+
```

### Pattern C

The canonical Hugo skill template, validated 5 times across V1 shipped features:

For any specialist capability, all deterministic mechanical work goes in a **single MCP tool call**. Hugo's role is to:

1. Invoke the tool once with the right parameters
2. Read the tool's return value
3. Synthesize the customer-facing summary from that return value

Hugo does not perform multi-step procedural work inline. Hugo does not "remember" what the tool did in a previous turn. Hugo does not infer state from prior session memory.

**The tool either returned a result or it didn't. Hugo synthesizes from the result.**

### Why it works

The v1 failure mode required soft preconditions to fail on. Pattern C eliminates them.

There's no "ask the user first" escape valve because the tool doesn't have a question-asking step. There's no "fabricate plausibly" path because the synthesis source is a concrete return value, not a probability distribution over plausible inputs. There's no "check session memory" shortcut because the agent's role is explicitly to summarize the latest tool return.

The model still chooses *whether* to call the tool, *which* tool to call, and *how* to phrase the customer summary. But the deterministic work — extract URLs from a brand context profile, fetch a page, write a markdown artifact, query an API, post via Upload-Post — happens in Python, not in the model.

### SKILL.md — the six canonical sections

Every Hugo SKILL.md follows the same six-section structure:

1. **Signature triggers** — mechanical phrase list (not intent-based). The `description` field drives Hermes auto-skill-selection against 87+ built-ins; without strong keyword overlap, Hermes picks a built-in over the Hugo skill.
2. **Numbered tiers** — 3-tier autonomy (Tier 1 = autonomous internal work; Tier 2 = bounded-spend with notice; Tier 3 = requires APPROVE).
3. **GOOD / BAD examples** — concrete, side-by-side. Bad examples are real failures from previous skill iterations; good examples are corrected patterns.
4. **Counterfactual** — *"if no `<tool_name>` result in this turn's tool history, `<step>` has not run; do NOT compose `<next step>`."* The hard gate. Without it, the model synthesizes plausibly anyway.
5. **Forbidden-tokens guardrail** — list of phrases Hugo must never emit in the customer channel (model names, internal paths, tool-progress text, framework names). The customer talks to Hugo, not to a stack.
6. **State-readback rule** — *"Before answering 'did you finish?' or 'what did you find?' follow-up, re-invoke the same MCP tool with `mode: "readback"` to retrieve the on-disk artifact; do not replay from session memory and do not read the file directly."*

### MCP server scaffolding

Every Hugo MCP server lives at `~/.hermes/mcp-servers/<domain>/server.py` with a `run.sh` wrapper.

**PEP 723 inline dependencies.** Each server declares its dependencies in a `# /// script` block at the top of `server.py`:

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

`uv run --script` reads this natively. No `pyproject.toml`, no `setup.py`.

**The `run.sh` shim.** Hermes registers MCP servers via `hermes mcp add <name> --command bash --args <path/run.sh>`. The shim exists because Hermes's `--script` argparse flag collides with `uv run --script`:

```bash
#!/usr/bin/env bash
exec uv run --script "$(dirname "$0")/server.py" "$@"
```

**Area-scoped server naming.** Servers are named for their functional area, not for the single skill they expose at creation time. A server named `planning/` holds `compose_plan` today and will hold `revise_plan` tomorrow. Don't create one server per skill.

### State-readback via `mode: "readback"`

When the customer asks again about a previously-generated artifact ("did you finish the audit?"), Hugo re-invokes the same MCP tool that produced it, passing `mode: "readback"`. The tool reads the artifact from disk and returns its contents. Hugo synthesizes the customer-facing summary from that return value.

Hugo does NOT read the artifact file directly (Hugo doesn't hold the absolute artifact path — that's internal to the tool). Hugo does NOT replay from session memory. Hugo does NOT make up the summary.

This closes the state-fabrication failure class. Every state query stays grounded in a live tool call.

### Hugo-scoped env var discipline

Every external API integration uses Hugo-scoped env var names — never the vendor SDK's default pickup name:

```python
# Correct — explicit, Hugo-scoped
genai.configure(api_key=os.environ["GOOGLE_AI_PRO_API_KEY"])

# Wrong — silently picks up GOOGLE_API_KEY from environment
genai.configure()
```

The rationale: a host machine may have global vendor env vars set for unrelated purposes. If a Hugo MCP server relied on the SDK's default env-var pickup, it would silently pick up the wrong key and produce authenticated-but-wrong API calls with no error surfaced. Three places, consistent naming: the `.env` file, the `os.environ[]` call, the install script's secret-collection step.

### Validation count

Pattern C has been validated five times across V1 shipped features.

| Skill | Server | Origin issue |
|---|---|---|
| `post-ad-hoc` | Upload-Post shell wrapper | #97 |
| `presence-audit` | `~/.hermes/mcp-servers/audit/` | #100 |
| `post-from-content-plan` | `~/.hermes/mcp-servers/content-plan/` | #102 |
| `reporting-lead` | `~/.hermes/mcp-servers/reporting/` | #104 |
| `compose-plan` | `~/.hermes/mcp-servers/planning/` | #83 (Pattern C origin) |

Each one follows the same shape: SKILL.md with the six canonical sections, MCP server with PEP 723 + `run.sh`, agent invokes the tool once and synthesizes the customer summary from the return.

---

## Enterprise topology · per-pod detail

The full per-pod reference architecture — every tier, every integration, every security boundary. The v2 architecture diagram above shows the *shape*; this one shows what's actually running where, layer by layer.

```
==============================================================================
CUSTOMER INTERFACE LAYER
==============================================================================

+----------------------------------+  +----------------------------------+
| CHANNELS · per tenant            |  | NETWORK EDGE                     |
+----------------------------------+  +----------------------------------+
|   Telegram (default)             |  |   Cloudflare Tunnel              |
|   Slack · WhatsApp               |  |     public ingress for OAuth     |
|   Signal · Email                 |  |     callbacks and webhooks       |
|                                  |  |                                  |
|   allowlist: only owner          |  |   Tailscale                      |
|     chat_id can DM the pod       |  |     admin-only SSH overlay       |
|   inline-button APPROVE flow     |  |     key-based, device allowlist  |
+----------------------------------+  +----------------------------------+

                                |
                                v

==============================================================================
TENANT POD · Docker container · one per customer · isolated runtime
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| PROCESS SUPERVISION  | | CHANNEL ADAPTER      | | HUGO AGENT RUNTIME   |
+----------------------+ +----------------------+ +----------------------+
|   systemd / launchd  | |   Hermes built-in    | |   single-agent       |
|     hermes-tenant    | |   outbound bot       | |     topology         |
|     -<id>.service    | |   APPROVE recognizer | |   binds bot token    |
|   Docker daemon      | |   message in/out     | |   skill picker       |
|     one container    | |   voice transcribe   | |   tool dispatcher    |
|     per tenant       | |   file uploads       | |     (Pattern C)      |
|   cron               | |                      | |   summarizer         |
|     scheduled tasks  | |                      | |     tool return ->   |
|     healthchecks     | |                      | |     customer reply   |
+----------------------+ +----------------------+ +----------------------+

+----------------------+ +----------------------+ +----------------------+
| BRAIN (CODEX CLI)    | | SKILLS LIBRARY       | | MCP TOOL SERVERS     |
+----------------------+ +----------------------+ +----------------------+
|   model:             | |   ~/.hermes/skills/  | |   ~/.hermes/         |
|   gpt-5.4-mini       | |     hugo/            | |     mcp-servers/     |
|   shell-out per turn | |   10 skills total    | |     <domain>/        |
|                      | |   5 shipped          | |                      |
|   per-pod ChatGPT    | |   5 in porting queue | |   8 area-scoped:     |
|   Plus subscription  | |                      | |     marketing__      |
|     customer's       | |   SKILL.md format:   | |     creative__       |
|     account          | |     1 triggers       | |     web__            |
|                      | |     2 numbered tiers | |     inbox__          |
|   rate-limit shared  | |     3 GOOD/BAD ex    | |     outbound__       |
|     across all turns | |     4 counterfactual | |     analytics__      |
|     in this pod      | |     5 forbidden tok  | |     audit__          |
|                      | |     6 state-readback | |     content_plan__   |
|   no Anthropic       | |                      | |                      |
|   no Gemini          | |   auto-allowlist via | |   scaffolding:       |
|                      | |   sync script        | |     Python + PEP 723 |
|                      | |                      | |     uv run --script  |
|                      | |                      | |     run.sh shim      |
+----------------------+ +----------------------+ +----------------------+

+----------------------+ +----------------------+ +----------------------+
| HITL GATE            | | STATE + AUDIT        | | SECRETS · CONFIG     |
+----------------------+ +----------------------+ +----------------------+
|   code-enforced      | |   ~/.hermes/         | |   tenants/<id>/      |
|   (not prompt-based) | |     sessions.db      | |     config/.env      |
|                      | |     (SQLite,         | |   chmod 600          |
|   Tier 1  auto       | |      per-pod)        | |   never committed    |
|     internal work    | |                      | |                      |
|   Tier 2  capped     | |   ~/.hermes/runlog/  | |   BOT_TOKEN          |
|     bounded reach    | |     audit trail      | |   OWNER_CHAT_ID      |
|   Tier 3  HITL       | |                      | |   CHATGPT_AUTH       |
|     APPROVE required | |   tenants/<id>/      | |   UPLOAD_POST_       |
|                      | |     context/         | |     API_KEY          |
|   APPROVE recognizer | |     BCP.md           | |   UPLOAD_POST_       |
|     phrase list      | |     plans/           | |     PROFILE          |
|                      | |     audits/          | |   GOOGLE_AI_PRO_     |
|   Forbidden tokens   | |     reports/         | |     API_KEY          |
|     never emit       | |     posts/           | |   GOOGLE_ADS_OAUTH   |
|     model / path /   | |     goals.md         | |   META_ADS_OAUTH     |
|     framework names  | |                      | |   SENDGRID_API_KEY   |
|                      | |                      | |   GA4_OAUTH          |
|                      | |                      | |   STRIPE_API_KEY     |
+----------------------+ +----------------------+ +----------------------+

  + tenant-2 pod · identical layout · isolated runtime
  + tenant-3 pod · identical layout · isolated runtime
  + ... (one Docker container per customer · no shared runtime state)

                                |
                       outbound API calls per
                       tool invocation
                                v

==============================================================================
EXTERNAL INTEGRATIONS · per-pod outbound
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| PER-CUSTOMER         | | HUGO-ORG             | | CUSTOMER-OWNED       |
| SUBSCRIPTIONS        | | SUBSCRIPTIONS        | | RESOURCES            |
+----------------------+ +----------------------+ +----------------------+
|   ChatGPT Plus       | |   Upload-Post Basic  | |   Google Ads account |
|     Codex inference  | |     managed-OAuth    | |     (customer's)     |
|                      | |     10 platforms     | |   Meta Ads account   |
|   Google AI Pro      | |     IG inbound API   | |     (customer's)     |
|     Veo 3 · Imagen   | |     per-tenant JWT   | |   Google Business    |
|     Nano Banana      | |                      | |     Profile          |
|                      | |   Cloudflare         | |                      |
|   Google Ads         | |     Tunnel + Pages   | |   Social profiles:   |
|     manager OAuth    | |                      | |     TikTok · IG · YT |
|   Meta Ads           | |   Tailscale          | |     LinkedIn · FB    |
|     manager OAuth    | |     admin SSH only   | |     X · Threads      |
|                      | |                      | |     Pinterest        |
|   SendGrid           | |   Hermes runtime     | |     Reddit · Bluesky |
|     outbound email   | |     MIT · no fee     | |                      |
|                      | |                      | |   (Hugo operates via |
|   GA4 · Stripe       | |   VPS host           | |    manager OAuth ·   |
|     read-only pulls  | |     multi-tenant     | |    customer owns)    |
|                      | |                      | |                      |
|   (customer billed   | |   (Hugo billed,      | |                      |
|    direct)           | |    amortized)        | |                      |
+----------------------+ +----------------------+ +----------------------+

==============================================================================
ADMIN · DATA SPINE  (Hugo-org · not per-tenant)
==============================================================================

+----------------------+ +----------------------+
| AIRTABLE             | | GITHUB               |
+----------------------+ +----------------------+
|   CRM                | |   Project board:     |
|   Activity log       | |     Hugo - Build     |
|   Per-tenant         | |     issues +         |
|     workspace        | |     decisions        |
|                      | |                      |
|   Joao admin only    | |   Private repo:      |
|                      | |     Hugo production  |
|                      | |     code             |
|                      | |                      |
|                      | |   PRD amendments log |
|                      | |     canonical        |
|                      | |     decision history |
+----------------------+ +----------------------+

==============================================================================
SECURITY BOUNDARIES
==============================================================================

+----------------------+ +----------------------+ +----------------------+
| ACCESS CONTROL       | | ISOLATION            | | GUARDRAILS           |
+----------------------+ +----------------------+ +----------------------+
|   Telegram allowlist | |   Per-tenant Docker  | |   Code-enforced      |
|     owner chat_id    | |     container        | |     spend caps in    |
|     only             | |   Per-tenant         | |     MCP wrappers     |
|                      | |     ~/.hermes/       | |     (not in prompts) |
|   OAuth scoping      | |   Per-tenant SQLite  | |                      |
|     manager-level    | |   Per-tenant         | |   Forbidden-tokens   |
|     (ops not billing)| |     Airtable         | |     prevents leaking |
|                      | |     workspace        | |     model / path /   |
|   Secret mgmt        | |                      | |     framework names  |
|     .env per pod     | |   No shared runtime  | |     in customer      |
|     chmod 600        | |     state            | |     channel          |
|     never in git     | |   No shared          | |                      |
|                      | |     filesystem       | |   State-readback     |
|   No ad spend on     | |                      | |     discipline:      |
|     Hugo's books     | |   Hermes ports       | |     re-invoke tools  |
|     (customer pays   | |     bound to         | |     (mode:           |
|      direct)         | |     127.0.0.1        | |      "readback")     |
|                      | |   Cloudflare Tunnel  | |     no session-memory|
|                      | |     only public      | |     replay · no      |
|                      | |     ingress          | |     direct file reads|
|                      | |   Tailscale-only SSH | |                      |
+----------------------+ +----------------------+ +----------------------+
```

**Three things to highlight from this layout:**

**Per-pod isolation is total.** Separate Docker container, separate `~/.hermes/` data dir, separate SQLite memory file, separate Airtable workspace, separate `.env` secrets file. Client A's brand context, plans, audits, and reports cannot enter Client B's prompt window — there's no shared filesystem path between them.

**The customer subscriptions belong to the customer.** ChatGPT Plus, Google AI Pro, Google Ads, Meta Ads, SendGrid, GA4, Stripe — all on the customer's card, all authed per-pod. Hugo operates them via manager OAuth; the bills never come to Hugo. Trade-off: gives up margin on those line items. Gains: no PCI scope, no AR risk, no payment-processing surface.

**The secrets tier is the only mixing point.** Every other layer is cleanly customer-owned or Hugo-owned. The `.env` file is where the customer's API keys (ChatGPT, Google AI Pro, ad-account OAuth tokens) sit next to the Hugo-org credentials (Upload-Post API key, Telegram bot token). `chmod 600` + per-pod isolation + never-committed-to-git is the boundary that keeps that mixing safe.

---

## The customer-facing surface

The internal architecture changed completely. The customer experience didn't.

### Channel

Telegram default; Slack, WhatsApp, Signal, Email as alternates configured per customer at install. The choice is per-customer because non-tech SMB owners need Telegram, tech-savvy ones often prefer Slack, and some have hard requirements on WhatsApp. Hermes provides the multi-channel adapter layer; the install script picks one per tenant.

### Approval gates

Three tiers of action autonomy, code-enforced (not prompt-enforced):

- **Tier 1 (auto)** — internal work: planning, audits, drafts, reports. No spend, no public surface.
- **Tier 2 (auto-capped)** — bounded spend or reach: weekend bid caps, posting cadence within a configured frequency. Caps are hardcoded in MCP tool servers, not in SKILL.md prompts.
- **Tier 3 (HITL)** — anything new and public: campaign launches, landing pages going live, outbound email send, social posts. Requires explicit "APPROVE" reply from the owner in the channel.

Customer always sees Tier 3 actions as a confirmation prompt:
> "Campaign ready. Landing page [URL] · ad previews [images] · forecast [budget] · reply APPROVE to launch"

### Weekly cadence

- **Monday** — "Here's what we should focus on this week" — Hugo posts the week's plan derived from `compose-plan` output.
- **Midweek** — opportunistic: *"I noticed X"* or *"I need approval for Y."* Only fires when there's something to flag.
- **Friday** — "Here are the results, spend, and next actions" — `reporting-lead` output.
- **Monthly** — plan revision: Hugo proposes adjustments to the multi-week plan based on the prior month's audit + reports.

### Customer's money stays on customer's accounts

A clean architectural decision: client ad spend is paid directly by the client on their own Google / Meta accounts. Hugo has manager-OAuth access (operates the campaigns), but doesn't hold or move the money. No float, no AR risk, no payment-processing surface, no "we'll bill you for the ads we ran" debate. **Trade-off:** gives up margin on ad-spend management. **Gains:** clean operating model, zero collections work, no PCI scope.

---

## Architecture diagrams

All diagrams in this repo are inline text-style ASCII inside fenced code blocks — readable on GitHub, version-controlled, diff-able, no rendering tooling required.

- **v1 team topology** — the 7-agent hierarchical team (Director + 6 specialists). See [The team](#the-team) above.
- **v2 single-agent topology** — Hugo + 8 area-scoped MCP tool namespaces. See [Architecture](#architecture) under the v2 section above.
- **Enterprise topology · per-pod detail** — full layered view: customer-facing layer · network edge · tenant pod · brain/inference · skills library · MCP servers · HITL gate · state/audit · secrets · external integrations · admin/data spine · security boundaries. See [Enterprise topology · per-pod detail](#enterprise-topology--per-pod-detail) above.

---

## Comparison vs alternatives

The marketing problem Hugo solves, framed against the human alternatives an SMB would consider:

| Option | Typical cost | What you get | Where it falls short |
|---|---|---|---|
| In-house marketer | €2,500–4,500/mo | One full-time person | Can't cover every channel well; turnover risk |
| Fractional CMO | €3,000–8,000/mo | Senior strategy + light execution | Strategy good, day-to-day execution thin |
| Digital agency | €2,000–10,000/mo | Coverage across channels | Often slow, project-based, high coordination cost |
| Hugo | (subscription) | Proactive AI marketer in your messaging app, with approval gates | Bounded by what's been ported (5 skills live, 5 in porting queue) |

(Pricing for alternatives: industry-typical ranges. Hugo's pricing is on the [live site](https://jccosta94.github.io/hugo-website/).)

The interesting structural property: **Hugo's marginal cost per customer is the per-customer subscription floor** (ChatGPT Plus + Google AI Pro + amortized Upload-Post + VPS share). Not the agency cost structure where team labour is the dominant variable. This is the architectural payoff of the single-agent + Pattern C topology — it scales by spinning up another tenant pod, not by hiring another marketer.

---

## What I designed vs what the platforms provide

For clear credit attribution:

| Concern | Platform provides | My architectural contribution |
|---|---|---|
| **Agent runtime** | Hermes Agent: SQLite session memory, multi-channel adapters, skill auto-selection, scheduled cron, Docker subagent sandboxing | Single-agent topology configured for the multi-tenant case · per-tenant Docker isolation pattern · `install-tenant.sh` provisioning shape · Telegram-allowlist binding |
| **Inference** | OpenAI Codex CLI: shell-out interface, ChatGPT subscription auth path | Picked the per-customer subscription model (ChatGPT Plus, not direct API) · verified `gpt-5.4-mini` as the accepted model id on the ChatGPT-account auth path (gotcha: `gpt-5-mini` and `gpt-5-codex` are rejected) · accepted the per-customer ToS-grey-area trade-off |
| **MCP tooling** | MCP protocol + Python SDK | Designed **Pattern C** as the canonical skill template · authored 8 area-scoped MCP server namespaces · PEP 723 + `run.sh` shim pattern · `mode: "readback"` state-retrieval convention · Hugo-scoped env-var discipline |
| **Social posting** | Upload-Post.com: managed-OAuth, 10-platform write, IG inbound | Picked Upload-Post over BYOK / Postiz / Outstand (after smoke-testing each) · designed per-tenant profile mapping via JWT connect URLs · webhook-driven `upload_completed` integration |
| **Creative gen** | Google AI Pro: Veo 3, Imagen, Nano Banana SDKs | Per-tenant subscription model (one Google AI Pro per customer) · `creative__*` MCP server wrapping the generation calls · brand-context-driven prompt assembly |
| **Approval gate** | Neither platform ships this | 3-tier autonomy taxonomy · code-enforced caps in MCP server wrappers (not in prompts) · APPROVE-phrase recognizer logic in Hugo's skill set · forbidden-tokens guardrail in every SKILL.md |
| **Audit / state** | Hermes session memory; per-tenant filesystem | `tenants/<id>/context/` structure (BCP · plans · audits · reports · post sidecars) · state-readback via tool re-invocation (not file read) |
| **Customer onboarding** | Neither platform ships this | `install-tenant.sh` provisioning script · BCP authoring flow via Telegram drop · per-tenant `.env` secret pattern · brand-context-profile extraction |

The platforms are the heavy lifting; **Pattern C, the single-agent topology, the per-tenant pod design, and the approval-gate / cadence orchestration are mine.**

---

## Stack

| Layer | v1 (multi-agent) | v2 (single agent + MCP) |
|---|---|---|
| Agent runtime | Hermes Agent (initial) → OpenClaw (multi-agent attempt) | Hermes Agent (MIT, Nous Research) |
| Topology | 7 agents (Director + 6 specialists) | 1 agent + 8 MCP tool namespaces |
| Inference | `gpt-5.4-mini` per agent, all on per-customer ChatGPT Plus | Same |
| Tool layer | Specialist agents' system prompts | Python MCP servers (PEP 723 inline deps + `uv run --script` + `run.sh` shim) |
| Owner channel | Telegram (default) · Slack / WhatsApp / Signal / Email | Same |
| Social posting | Upload-Post.com (managed OAuth, 10 platforms) | Same |
| Creative gen | Google AI Pro (Veo 3 + Imagen + Nano Banana) per customer | Same |
| Web hosting | Cloudflare Pages | Same |
| Infrastructure | Cloud VPS · Docker per tenant · Tailscale (admin SSH) · Cloudflare Tunnel (ingress) | Same |
| Memory | Per-tenant SQLite + tenant workspace filesystem | Same |
| HITL gates | 3 autonomy tiers, code-enforced | Same |
| CRM / activity log | Airtable | Same |

**The v1 → v2 transition didn't change a single externally-visible component.** Only the internal topology and the canonical skill template changed.

---

## Deeper reading

- [`case-study/architecture-evolution.md`](./case-study/architecture-evolution.md) — the v1 → v2 narrative in long form: where we started, the hosting and brain journey (Mac mini → VPS, Anthropic → Codex, Hermes → OpenClaw → Hermes), the collapse, what stayed and what changed, what I learned.
- [`orchestration/mcp-integration.md`](./orchestration/mcp-integration.md) — Pattern C as a technical reference: the six canonical SKILL.md sections, MCP server scaffolding (PEP 723 + `run.sh`), state-readback via `mode: "readback"`, env-var discipline, validation count, what Pattern C is *not* good for.
- *(planned)* `case-study/business-problem.md` — why an AI marketing employee for SMBs, not a tool.
- *(planned)* `case-study/orchestration-design.md` — the Telegram-default decision, autonomy tier design, weekly cadence.
- *(planned)* `case-study/ai-stack.md` — the brain/model journey (Anthropic Max → Codex CLI → split with Gemini → all-Codex).
- *(planned)* `case-study/lessons-learned.md` — what I'd do differently.
- *(planned)* `partial-code-examples/` — sanitized SKILL.md, MCP server, install script.

---

## Status

- 🛑 **v1 (multi-agent team)** — never shipped to customers. Three delegation failures (Issues #83, #81, #86) under `gpt-5.4-mini` triggered the architectural pivot before V1 reached production. Specialist agent definitions archived as reference material under `agent-drafts/`.
- ✅ **v2 (single agent + Pattern C)** — current architecture. **5 skills shipped:** `post-ad-hoc`, `presence-audit`, `post-from-content-plan`, `reporting-lead`, `compose-plan`. **5 skills in porting queue:** `ads-manager`, `website-builder`, `brand-designer`, `inbox-reviews-triage`, `outreach-assistant`.
- ✅ **Live site** — [jccosta94.github.io/hugo-website](https://jccosta94.github.io/hugo-website/) describes the customer-facing offering in plain language.
- 🚧 **First customer** — paint manufacturer pivoting B2B → consumer DIY, onboarding in progress.
- ✅ **Diagrams** — all architecture diagrams shipped as inline text-style ASCII in fenced code blocks (v1 team topology · v2 single-agent topology · enterprise per-pod detail).
- 🚧 **Sub-docs** — `case-study/architecture-evolution.md` and `orchestration/mcp-integration.md` complete. Workflows, comparisons, examples folders TBD.

---

## Acknowledgements

The platforms this case study is built on:

- **[Hermes Agent](https://hermes-agent.nousresearch.com/)** by [Nous Research](https://nousresearch.com) — the open-source autonomous agent framework (MIT) that Hugo runs on. Native multi-channel adapters, persistent SQLite memory, scheduled cron, Docker subagent sandboxing.
- **[OpenClaw](https://openclaw.ai/)** by [@steipete](https://x.com/steipete) — the platform where the v1 multi-agent design was attempted before the collapse to Pattern C. Also the platform underpinning [Psinest's build pipeline](../openclaw-hermes-evolution/) (independent project with its own case study).
- **[Codex CLI](https://platform.openai.com/)** — the OpenAI Codex CLI (`gpt-5.4-mini`) that handles all Hugo inference under per-customer ChatGPT Plus subscriptions.
- **[Upload-Post.com](https://upload-post.com)** — managed-OAuth social posting layer (10 platforms outbound + Instagram inbound), which eliminated the per-platform BYOK + tier-gating burden.
- **[Google AI Pro](https://gemini.google.com)** — Veo 3, Imagen, and Nano Banana for creative generation, per-tenant subscription model.

**Pattern C** is mine — the canonical Hugo skill template (single MCP tool call + summarize-the-return), developed across Issues #83, #97, #100, #102, #104 and codified after the third multi-agent delegation failure showed that fixing the topology was a shorter path than fixing the model. The six-section SKILL.md structure (signature triggers · numbered tiers · GOOD/BAD examples · counterfactual · forbidden-tokens · state-readback) is the operational form.

The customer-facing UX (one chat-based marketing employee, weekly cadence, APPROVE-gated public actions) is borrowed from how a real marketing employee operates with a small-business owner. **Single-agent systems work best when they mirror the practices that already work for one human in that role.**

---

📧 [jccosta94@gmail.com](mailto:jccosta94@gmail.com) · 🌐 [jccosta94.github.io/hugo-website](https://jccosta94.github.io/hugo-website/)
