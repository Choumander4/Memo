# Multi-Agent Router & Orchestration Architecture

A build reference for reproducing this system's **Router → Orchestrator → Sub-Agent**
design elsewhere. It documents what exists today in [router.py](/c:/Users/haminhch/OneDrive - Intel Corporation/PM Chatbot/nuvia_shared/tegaf_home/teg_ai_agent/agents/router.py)
and [sub_agents.py](/c:/Users/haminhch/OneDrive - Intel Corporation/PM Chatbot/nuvia_shared/tegaf_home/teg_ai_agent/agents/sub_agents.py),
plus enough of the surrounding system (data models, async execution) for the
pattern to make sense end-to-end.

## 🗺️ Start Here: The Whole System in One Minute

Think of this like a **help desk with a front-desk receptionist and four
specialists sitting behind them**:

- The **receptionist** (the *Router*) doesn't answer your question — they
  just listen for a few seconds and decide *which specialist* should take it.
  Most of the time they can tell from one or two obvious keywords in what you
  said ("downtime" → send to the downtime specialist). Only when it's a
  genuinely confusing request do they pause and think harder about it.
- The **office manager** (the *Orchestrator*) is the one who actually walks
  you over to the specialist, hands them your question plus a quick recap of
  what you two already discussed today, and waits for their answer. If that
  specialist is suddenly unavailable (out sick), the manager immediately
  tries their backup instead of leaving you standing there.
- The **specialists** (the *Sub-Agents*) are the ones who actually do the
  work — each has their own notebook of know-how (a prompt) and their own set
  of tools on their desk (downtime reports, a calculator, a phone to call the
  document archive, etc.).

**One example, start to finish:**

> You type: *"What's the downtime for CAM last shift?"*
>
> 1. The receptionist spots the word **"downtime"** → instantly sends you to
>    the **Tool Agent** (no deep thinking needed, it's an obvious case).
> 2. The office manager hands the Tool Agent your question, plus "you two
>    haven't talked about CAM before today."
> 3. The Tool Agent uses its **time tool** to figure out what "last shift"
>    means as an actual date/time range, then uses its **downtime-report
>    tool** to pull real numbers for module CAM in that window.
> 4. It writes up the answer as a neat report and hands it back.
> 5. The office manager passes that answer straight back to you, along with
>    a note of which tools were used — done.

Everything below is the detailed version of that story — how the
receptionist decides, how the manager handles retries and progress updates,
and what each specialist's notebook and toolset actually look like.

## Scope Notes & Assumptions

A few defaults were applied while writing this (since they couldn't be
confirmed live) — flag if any of these don't match what you want:

1. **The Technician/Document agent is covered only briefly.** It depends on
   a separate document-search system (RAG over manuals/guides) that's a
   bigger, separate build effort. Treat it as a placeholder for now.
2. **The Tool, Analytics, and General agents are covered in real depth** —
   their prompts, tools, models, and logic — since those can be built without
   extra infrastructure.
3. **The "LLM provider" parts are described generically** ("Provider A /
   Provider B") instead of naming specific vendors, since that's a swappable
   implementation detail.
4. **Model choices are described by role** (fast & cheap, standard, deep
   reasoning, reasoning-with-explanation) instead of literal model version
   names, since those change over time.
5. Includes small pseudocode snippets (not real source code) just to make
   the data shapes concrete.
6. The web-server/background-worker plumbing is described briefly for
   context, but the real focus — as requested — is the Router, Orchestrator,
   and Sub-Agents.

---

## Table of Contents

1. [Purpose & Design Goals](#purpose--design-goals)
2. [High-Level Flow](#high-level-flow)
3. [Layered Architecture](#layered-architecture)
4. [Request Lifecycle & Durable Execution](#request-lifecycle--durable-execution)
5. [The Router](#the-router)
6. [The Orchestrator](#the-orchestrator)
7. [Sub-Agent Framework (shared harness)](#sub-agent-framework-shared-harness)
8. [Tool Agent](#tool-agent)
9. [Analytics Agent](#analytics-agent)
10. [General Agent](#general-agent)
11. [Technician / Document Agent (brief)](#technician--document-agent-brief)
12. [Tool Inventory Summary](#tool-inventory-summary)
13. [Conversation Memory Strategy](#conversation-memory-strategy)
14. [Suggested Improvements / Alternative Approaches](#suggested-improvements--alternative-approaches)
15. [Build Checklist](#build-checklist)

---

## Purpose & Design Goals

This system is a chat assistant over manufacturing data (tool downtime,
alarms, output, maintenance tickets, equipment documentation). Every message
gets routed to exactly one of several **specialist sub-agents**, each with
its own prompt, tools, and model. In plain terms, it's built around five
goals:

- **Don't waste an LLM call just to decide who answers.** Most questions are
  obvious enough that a cheap keyword check can route them instantly — save
  the "thinking" model for the genuinely tricky ones.
- **Never let one flaky API call ruin a turn.** If the model provider you
  normally use has an outage or times out, silently retry the same request on
  a backup provider before giving up.
- **Remember the conversation without re-reading all of it every time.**
  Keep the last few messages in full, and boil everything older down into a
  short running summary — like taking notes instead of replaying a whole
  recording.
- **Don't tie up the web request while the AI is thinking.** Hand the actual
  work off to a background worker, and let the user's screen check in on
  progress rather than sit there waiting on one open connection.
- **Build every specialist the same way.** Same recipe every time (a prompt
  + a list of tools + a model) so adding specialist #5 later is mostly
  copy-and-fill-in, not a redesign.

One important nuance: this is a **"router hands off to one specialist"**
design, *not* a team of agents that talk to each other. The Orchestrator
picks one specialist, that specialist answers, and that's the whole turn —
no specialist can currently ask another specialist for help mid-answer. (See
[Suggested Improvements](#suggested-improvements--alternative-approaches) for
when that might be worth changing.)

---

## High-Level Flow

In short: **a message comes in, gets classified once, goes to one specialist,
and the answer flows back out** — with progress updates along the way. Here's
that same flow as a diagram:

```
 User message
      │
      ▼
┌─────────────────────┐
│  Presentation layer  │   (web endpoint: receive message, persist it,
│                      │    create a "run" record, return immediately)
└──────────┬───────────┘
           │ enqueue
           ▼
┌─────────────────────┐
│  Async execution     │   (background worker picks up the run)
│  layer               │
└──────────┬───────────┘
           │ query + context + conversation_id
           ▼
┌─────────────────────────────────────────────────────────────┐
│                        ORCHESTRATOR                          │
│  1. Load conversation history (recent + rolling summary)     │
│  2. Classify query → pick ONE sub-agent                      │
│  3. Directly invoke that sub-agent (primary model)           │
│     └─ on exception → retry once on fallback model           │
│  4. Stream progress events (routing/tool_start/tool_end/done)│
└──────────┬────────────────────────────────────────────────────┘
           │ dispatch
           ▼
┌─────────────────────┐        ┌───────────────────────────┐
│   SUB-AGENT          │──────▶│  Tools (local Python funcs, │
│ (Tool / Analytics /  │       │  and/or external MCP tools) │
│  General / Technician)│      └───────────────────────────┘
└──────────┬───────────┘
           │ final answer + tools_used + raw_tool_outputs
           ▼
┌─────────────────────┐
│  Async execution     │  persists progress at every step,
│  layer               │  saves the final assistant message
└──────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│  Presentation layer  │  client polls (or streams) until done
└─────────────────────┘
```

---

## Layered Architecture

Each row below is one job in the pipeline — read it top to bottom in the
order a message actually flows through it:

| Layer | Responsibility | Current implementation |
|---|---|---|
| Presentation | Accept user message, return immediately, expose poll/cancel endpoints | Django views |
| Async execution | Run the orchestrator off the request thread; persist progress; enable cancellation | Celery task + a durable "run" DB row, polled by the client |
| **Orchestrator** | Classify the query, pick a sub-agent, run it with provider fallback, emit progress events | `RouterOrchestrator` in `router.py` |
| **Router (classifier)** | Decide which sub-agent handles a given query | Keyword fast-path + LLM structured-output fallback, both in `router.py` |
| **Sub-agents** | Domain-specific reasoning + tool use | `tool_agent`, `general_agent`, `analytics_agent`, `technician_expert_agent` in `sub_agents.py` |
| Tools | Concrete data access / actions an agent can call | Local Python functions (DB queries, SQL, calendar math, knowledge-graph lookups) + external MCP tool server (document search) |
| Model/Provider | LLM access with tiering + redundancy | Multiple model configs per task role, each with a primary + fallback provider |
| Persistence | Conversations, messages, run/progress state | Conversation / Message / Run tables |

---

## Request Lifecycle & Durable Execution

High-level context only — not the focus of this doc, but useful for seeing
where the Orchestrator fits in the bigger picture. In short: **the web page
never waits on the AI directly — it hands the work off and checks back.**

1. **Presentation layer** receives the message, saves it, creates a **Run**
   record (`pending`, 0%), and hands the work off in the background. It
   replies immediately with a "thinking…" placeholder — the web request
   doesn't sit there waiting for the LLM.
2. **Async execution layer** (a background worker) picks up the Run, marks it
   `running`, and calls the Orchestrator's **progress-event generator**
   (`iter_progress_events`). Every event it produces updates that same Run
   row (status text, percent, current tool, a running list of steps), so
   whatever the client is polling always reflects live progress.
3. When the generator produces its final `done` event, the worker saves the
   finished assistant message (with which tools were used, their raw
   outputs, and a step-by-step trace for debugging) and marks the Run
   `completed`.
4. On `error`, the Run is marked `failed` and an error message is saved
   instead.
5. **Cancellation**: the client can flag a Run as `canceled`; the loop
   consuming events checks for that flag between events and stops early
   (which also cancels the agent work still in flight).
6. **Fallback path**: if the background queue itself is unavailable, the
   exact same Orchestrator call can run synchronously in the web process
   instead — same logic, just no live progress updates, like a plain
   "call it and wait" version.

**Why this matters:** the Run row in the database — not an open browser
connection — is the single source of truth for "what's happening with this
message." That's what makes it survive a dropped Wi-Fi connection or a
worker restart without losing the user's answer.

---

## The Router

The Router has exactly one job: **given a query + conversation history + any
special context flags, pick exactly one specialist to send it to.**

### Classification schema (contract)

```text
Classification = { source: one of ["general_agent", "tool_agent",
                                    "analytics_agent", "technician_expert_agent"] }

RouterState = {
  query:                str
  context:               dict   # UI-supplied filters: module, entity, user identity, feature flags
  conversation_history:  list   # prior turns as role-tagged messages
  classifications:       list[Classification]
  results:                list  # accumulated sub-agent outputs
}
```

### Decision order (first match wins)

Think of this as a checklist the receptionist runs through, top to bottom,
stopping at the first thing that matches:

1. **Forced routes from context flags** — e.g. a "document mode" toggle in
   the UI always forces the Technician agent, no matter what the words say.
   These are checked first and skip classification entirely: zero ambiguity,
   zero cost.
   > *Example: the user has "Search Documents" toggled on in the sidebar,
   > then asks "how do I calibrate this?" → straight to Technician agent,
   > no keyword or LLM check needed.*
2. **Keyword fast-path** (a regex check on the raw text — no LLM call at
   all):
   - Root-cause / "why" language → **Analytics agent** (it has the causal
     reasoning tools). Checked first so it isn't stolen by a bare downtime
     keyword.
     > *Example: "why was CAM down for 6 hours yesterday?" → contains "why"
     > → Analytics agent, even though it also mentions downtime.*
   - Plain downtime / missing-report language → **Tool agent**.
     > *Example: "what's the downtime for CAM last shift?" → Tool agent.*
   - Other data-analysis language (alarms, tickets, output, "list/show/give
     me…") → **Analytics agent**.
     > *Example: "how many alarms this week?" → Analytics agent.*
   - Troubleshooting/maintenance/manual language → **Technician agent**.
     > *Example: "how do I replace the sensor on JDC001?" → Technician agent.*
   - No match at all → fall through to the LLM classifier.
3. **LLM classifier fallback** (only for genuinely ambiguous queries — the
   goal is that most traffic never reaches this step):
   - A small/fast model reads the query and returns a **structured**
     decision matching the schema above (using function-calling style
     structured output, which tends to be more compatible across gateways
     than asking for raw JSON).
   - It's told: pure greetings/small talk always go to **General agent**, no
     matter what was discussed earlier.
     > *Example: after a long conversation about alarms, the user just says
     > "thanks!" → General agent, not Analytics agent — small talk always
     > wins over history.*
   - It's also told how to handle **incomplete follow-ups**: resolve them
     using the previous turn's topic, *unless* the follow-up itself contains
     a new domain keyword, in which case the new keyword wins.
     > *Example A: previous turn was about CAM downtime; user now says "what
     > about last week?" (no new keyword) → stays on Tool agent (same topic,
     > new time range).*
     > *Example B: previous turn was about CAM downtime; user now says "what
     > about alarms?" (contains a new keyword, "alarms") → switches to
     > Analytics agent, because the new message itself signals a topic
     > change.*
   - If the classifier model call itself fails, it retries on a backup fast
     model, then (last resort) a backup full-size model — the system always
     needs to land on *some* agent, never silently gives up on routing.

### Why keyword-first?

A regex check is instant and free — it just needs to be right for the
*obvious* majority of cases. Anything genuinely unclear falls through to the
LLM instead of forcing a shaky keyword guess. In practice, getting the easy
70–80% right for free is what buys the latency budget to let the LLM
carefully handle the harder 20–30%.

---

## The Orchestrator

The Orchestrator owns everything *after* a route is picked: loading memory,
running the specialist, retrying on a backup provider if needed, and
reporting progress. Importantly, it does **not** re-run any complex
multi-agent graph — it classifies once, then calls that one specialist
function directly. Think "hand off and wait," not "loop between agents."

### Responsibilities

1. **Load conversation memory** — see [Conversation Memory Strategy](#conversation-memory-strategy).
2. **Classify** — run the Router logic above, once, per incoming message.
3. **Pick a model pair for that agent** — each specialist has a **primary
   model** and a **backup model** (see [model tiers](#model-tiers) below).
4. **Invoke with fallback**:
   ```text
   try:
       return invoke(agent, primary_model, state)
   except Exception:
       return invoke(agent, fallback_model, state)   # one retry, different provider
   ```
   Any failure at all — auth error, server error, timeout, rate limit —
   triggers exactly one retry on the other provider. It doesn't matter *why*
   the first one failed; the response is the same either way.
5. **Stream progress events** — the main entry point is a generator that
   yields small event dicts as things happen. Here's what a *real* turn's
   event stream might look like for the "downtime for CAM last shift"
   example from the top of this doc:

   ```text
   {"type": "status",  "content": "Reviewing request and selecting the best agent..."}
   {"type": "routing", "agent": "tool_agent", "model_version": "..."}
   {"type": "status",  "content": "Preparing agent execution..."}
   {"type": "tool_start", "tool": "get_ww_range", "input": "shift_offset=1"}
   {"type": "tool_end",   "tool": "get_ww_range", "output": "ww_from=..., ww_to=..."}
   {"type": "tool_start", "tool": "downtime_report_tool", "input": "module=CAM, ..."}
   {"type": "tool_end",   "tool": "downtime_report_tool", "output": "..."}
   {"type": "tool_usage_summary", "tools_used": ["get_ww_range", "downtime_report_tool"]}
   {"type": "done", "response": "### CAM Downtime — Shift ...\n...", "agent_type": "tool_agent", ...}
   ```

   The full set of event types:

   | Event type | Meaning |
   |---|---|
   | `status` | Human-readable status text (e.g. "selecting the best agent…") |
   | `routing` | Which agent + model tier was selected |
   | `thought` | Optional pre-tool-call reasoning text surfaced to the UI |
   | `tool_start` | A tool call began (name, input preview, running tool count) |
   | `tool_end` | A tool call returned (output preview, duration) |
   | `tool_usage_summary` | Final dedup'd list of tools used this turn |
   | `done` | Final answer + metadata (agent type, model tier, route path, tools used, raw tool outputs, timing) |
   | `error` | Something failed; carries an error message |

   Under the hood, a callback attached to the specialist's tool-calling loop
   pushes these events onto a queue while the specialist keeps working in the
   background — so the events above show up *while* the agent is still
   thinking, not all at once at the very end.
6. **Plain (non-streaming) entry point** — a simple `invoke(query, context,
   conversation_id) -> result_dict` also exists, for callers that just want
   the final answer with no live progress (e.g. the sync-fallback path).
   Same classify → invoke-with-fallback logic, just without the events.

### Model tiers

Rather than one model for everything, each specialist (and the classifier
itself) gets a model suited to its job — and every one of these has both a
primary and a backup provider behind it:

| Tier | Used by | Why |
|---|---|---|
| Fast / cheap, temperature 0 | Classifier, General agent | Routing and small talk don't need deep reasoning; speed matters most here |
| Standard reasoning | Tool agent | Structured tool calls over a fixed schema; moderate reasoning is enough |
| Deep reasoning | Technician agent | Needs to reason carefully over long document excerpts |
| Reasoning-with-visible-explanation | Analytics agent | Open-ended SQL/graph reasoning benefits from showing the user *why* it chose a certain path; also has its own cheap "just format this" variant for the tool-free summary mode |

One config switch picks which provider is primary vs. backup, system-wide —
so swapping providers is a config change, not a code change.

---

## Sub-Agent Framework (shared harness)

Every specialist is built and run the same way — this shared "recipe" is
what makes it easy to bolt on a 5th or 6th specialist later without
redesigning anything.

**Construction, per call:**

1. Start from a **static system prompt** for that agent (its role, domain
   vocabulary, tool-usage rules, output formatting rules).
2. **Inject dynamic context** into the prompt — the same static prompt gets a
   few extra lines appended based on what's true *right now*:
   - Active UI filters, e.g. if the user has module `CAM` selected in the
     sidebar, the prompt gains a line like: *"The user has filtered by
     modules: CAM. Focus your analysis and tool queries on this module
     unless they explicitly ask about others."*
   - A language rule: reply in whatever language the user wrote in, but
     never translate codes, IDs, or SQL.
   - If the user's name is known, a friendly "greet them by name" line.
3. **Resolve the tool list**: local Python tools for that agent, plus —
   if that agent is allowed to — tools fetched live from an external MCP
   tool server. A small per-agent registry controls this:
   - `None` → allow every MCP tool that server offers
   - empty set → skip connecting to MCP entirely (saves time for agents that
     don't need it, e.g. Tool/General agent)
   - a specific set of names → only allow those particular MCP tools
4. **Compile (or reuse) a tool-calling agent** for that (agent name, model)
   pair. This compiled agent is cached, so the same underlying react-agent
   isn't rebuilt on every single message — only swapping in a different
   model invalidates the cached copy. (MCP-backed agents are the one
   exception — they're rebuilt each time, since their tool list can change
   per connection.)
5. **Build the message list**: system prompt (with its injected context) +
   the last several turns of conversation history + the current question.
6. **Invoke** the compiled agent, attaching the progress-callback if the
   Orchestrator supplied one.
7. **Return a uniform result** — every specialist hands back the exact same
   shape, regardless of what it did internally:
   ```text
   AgentOutput = {
     source:            agent name
     result:             final answer text (visible text only — any hidden
                          "thinking" content is filtered out)
     tools_used:         [tool names called, in order]
     raw_tool_outputs:    { tool_name: raw_output_string, ... }
   }
   ```

**MCP connection fallback:** if the external tool server can't be reached
(say, its process crashed), the agent quietly falls back to its local tools
only, and tells the user "some tools are unavailable" rather than failing the
whole turn. (There's one extra wrinkle worth knowing but not necessarily
copying: if a background worker specifically can't reach that tool server
but the main web process can, it raises a special "retry me over there"
error so the turn gets handed back to a process that *can* reach it — a
nice-to-have, not a requirement.)

---

## Tool Agent

**Purpose:** answer *standard, clearly-scoped* downtime and missing-report
questions using direct database lookups — no open-ended reasoning, no
SQL-writing.

**Worked example:**
> User: *"downtime for CAM last shift"*
> 1. Calls the time-window tool with `shift_offset=1` → gets back concrete
>    dates/hours for "last shift."
> 2. Calls the downtime-report tool with `module=CAM` + that time window.
> 3. Renders the tool's own report format (headings per module → link →
>    entity → issue) — it does **not** rewrite this into a different-looking
>    summary table, even though that's its usual style elsewhere.

**Tools (local only — MCP is intentionally turned off here for speed):**
- A time-window tool: turns phrases like "last shift," "last 2 weeks," or
  "today" into concrete date/shift ranges in one call. It's the *primary*
  time tool; a couple of backup/legacy time tools exist for edge cases
  (an exact calendar date, or if the primary tool is unavailable).
- A downtime-report tool: returns a hierarchical report (module → link →
  entity → issue).
- A missing-report tool: for missing-report pulls.

**Prompt strategy highlights:**
- Teaches the model the domain's naming hierarchy (Module → Operation →
  Link → Entity) so it can correctly parse shorthand like "CAM001" or
  "DGX00003."
- **Won't guess if scope is unclear.** Before calling any tool, it needs both
  a *time* scope ("last shift," "last 2 weeks," etc.) and an *entity/module*
  scope. If either is missing, it asks one short clarifying question instead
  of assuming.
  > *Example: user just says "show me downtime" (no timeframe, no
  > module) → agent asks "Sure — for which module, and what time range?"
  > instead of guessing "last 24 hours, all modules."*
- **The tool's output format always wins.** Once the downtime tool runs, its
  built-in formatting rules override the agent's normal Markdown style — no
  "cleaning it up" into a different layout.

**Model:** the "standard reasoning" tier — enough to carefully pull out
parameters and sequence tool calls, without paying for the most expensive
tier.

---

## Analytics Agent

**Purpose:** the open-ended data-analysis specialist — alarms, tickets,
output/yield, custom aggregations, rankings, comparisons, and (its most
distinct trick) **figuring out *why* something happened** using a knowledge
graph of causal relationships.

**Worked examples:**
> **Standard question:** *"how many alarms this week?"*
> 1. Calls the schema tool to see what tables/columns are available.
> 2. Writes a SELECT query and runs it through the SQL-execution tool.
> 3. If the query errors, fixes it and tries again.
> 4. Interprets the rows and replies.

> **Root-cause question:** *"why was CAM down for 6 hours yesterday?"*
> 1. Instead of starting from the full schema, it first asks the
>    knowledge-graph tool for tables *causally linked* to "downtime" —
>    ranked by how relevant/close they are.
> 2. Feeds those specific tables into the schema tool (skipping the
>    unrelated ones).
> 3. Writes and runs SQL against just those tables, then explains its
>    reasoning path in the answer (which is why this agent gets a model tier
>    that narrates its own thought process — see below).
> 4. If the knowledge graph has no causal links for this particular symptom,
>    it just falls back to the standard full-schema approach above.

> **Tool-free "just format this" mode:** sometimes the caller (the web page)
> already has the data on hand (e.g. a pre-built summary widget) and just
> wants it turned into readable prose — no tools are called at all here, and
> a cheaper/faster model handles just the writing.

**Tools:**
- A schema tool: returns column-level details (type, nullable, description)
  for every table it's allowed to query — always called first in the
  standard path so the model knows what exists.
- A SQL-execution tool (read-only `SELECT`s only): runs the generated query
  and returns rows.
- Knowledge-graph tools: look up the relationship between two entities, find
  a path between two entities, and — the key one — return causally-ranked
  tables for a "why" question.
- An output/yield tool.

**Model:** the "reasoning-with-visible-explanation" tier — since this
agent's tasks are the most open-ended, showing *why* it picked a particular
table/join/root-cause path builds user trust in the answer.

---

## General Agent

**Purpose:** the default/catch-all for greetings, small talk, and anything
with no manufacturing/technical content in it. Deliberately the simplest
agent in the system.

**Worked example:**
> User: *"hey, how's it going?"* → routed here by the keyword/LLM router →
> replies conversationally, no tools called at all.

**Tools:** just calendar/time helpers (the same time-window tool used
elsewhere, plus a couple of date-conversion helpers) — no data-access tools.

**Prompt:** short and friendly — a helpful-assistant persona, uses
conversation history for continuity, standard Markdown formatting. No
clarification gates, no domain vocabulary to learn.

**Model:** the "fast & cheap" tier — this path should feel near-instant.

---

## Technician / Document Agent (brief)

**Purpose:** troubleshooting/maintenance Q&A grounded in equipment manuals
and best-known-methods (BKM) documents.

**Worked example:**
> User: *"how do I calibrate the alignment sensor on a DGX tool?"* → routed
> here → it searches the document knowledge base for relevant manual
> sections → answers using what it finds, citing which document/section it
> pulled from.

**Key difference from the other three:** its main tool is **document
search**, which lives on a **separate external server** (a different process
running its own document-retrieval pipeline) instead of being a local Python
function. Locally, it only has the same calendar/time helpers as the General
agent, as a fallback.

It also picks up extra UI filters (document type, document source) and folds
them into the prompt as active search scopes — with an instruction to tell
the user which filter might be narrowing results too much if a search comes
back empty or off-topic.

**Model:** the "deep reasoning" tier (careful multi-step reasoning over
retrieved document text).

**Build note:** this agent is only as good as the document search pipeline
behind it — and that pipeline (chunking documents, generating embeddings, a
searchable vector store, a retrieval tool) is a whole separate project. For
a first build, it's fine to stub this route with a placeholder reply like
"document search isn't set up yet" and come back to it once that pipeline
exists — don't let it block everything else.

---

## Tool Inventory Summary

A quick cheat-sheet of every tool, who uses it, and what it's for:

| Tool | Used by | Purpose |
|---|---|---|
| Time-window resolver (primary) | Tool, General, Technician | Relative time phrase → concrete date/shift range, in one call |
| Calendar / date-conversion (fallback) | Tool, General, Technician | Explicit date → domain time-period label |
| Legacy datetime tool (fallback) | Tool, General, Technician | Raw shift boundaries without labels |
| Downtime-report tool | Tool agent | Hierarchical downtime report (module → link → entity → issue) |
| Missing-report tool | Tool agent | Missing-report pulls |
| Schema-introspection tool | Analytics agent | Column-level schema for accessible analytics tables |
| SQL-execution tool | Analytics agent | Run a validated SELECT, return rows |
| Output/yield tool | Analytics agent | Output/yield metrics |
| KG relationship lookup | Analytics agent | Relationship between two entities |
| KG path finding | Analytics agent | Path between two entities |
| KG root-cause analysis | Analytics agent | Weight-ranked causal tables for a symptom |
| Document search (MCP) | Technician agent | RAG over manuals/BKMs (external process) |

---

## Conversation Memory Strategy

The goal: stay coherent across a long conversation *without* re-sending the
entire chat history (and its token cost) on every single message.

- Keep the **last few messages in full** (a small window covering the most
  recent exchanges in complete detail).
- **Summarize everything older** into a short 2–3 sentence running summary.
- That summary is **cached**, and only **regenerated occasionally** (roughly
  every time the older-message pile grows by another full window's worth) —
  not on every message, so you're not paying for a summarization call every
  single turn.
- Short conversations skip summarization altogether — if the whole
  conversation already fits inside the "recent" window, just use all of it.

> **Example:** with a recent-window of 6 messages, message #7 through #26
> come along as full text, and everything before message #7 gets folded into
> one short paragraph like *"User has been asking about CAM downtime and
> alarm counts over the past few days; no open issues yet."*

The loaded history feeds both the Router (so it can resolve follow-ups like
"what about last week?") and the chosen specialist (for continuity) — with
plain status/system notes stripped out before showing it to the Router, to
save tokens.

---

## Suggested Improvements / Alternative Approaches

Optional ideas, not requirements — included because the original request
invited a critique of the current design. Each one below follows the same
pattern: *what it is → why it might matter → when it's actually worth doing.*

1. **Let specialists collaborate, not just hand off once.**
   Today, the Orchestrator picks exactly one specialist per turn — there's
   no way for, say, the Analytics agent to ask the Tool agent for raw
   numbers mid-answer. If that kind of teamwork becomes a real need, a
   graph-style "supervisor + workers" pattern (agents that can hand off to
   each other, with shared notes) fits better than stretching the current
   simple "classify → invoke" function.
   *Worth it: only once you actually need cross-agent teamwork — it adds
   real complexity (shared state, knowing when to stop looping, more
   latency) that isn't justified pre-emptively.*
2. **Push updates instead of having the client keep asking.**
   Polling the database for progress is simple and very robust to dropped
   connections, but it costs a bit of latency and database load. A hybrid —
   push live updates over a socket to a connected browser, while still
   writing to the same durable record for anyone who reconnects — gets you
   the best of both.
   *Worth it: only once polling frequency/DB load is a measured, real
   problem — not by default.*
3. **Let the classifier say "I'm not sure" instead of always committing.**
   Right now it always outputs one confident label. Returning a confidence
   score (plus maybe a second-choice guess) would let a low-confidence case
   trigger a quick clarifying question instead of silently guessing.
   *Worth it: if wrong routing turns out to happen often enough to notice —
   otherwise it's unnecessary overhead.*
4. **Write the "try backup provider" logic once, not per agent.**
   The "try primary, on failure try backup" pattern is currently repeated
   for every agent/model pairing. A single shared helper function would cut
   down on repeated boilerplate.
   *Worth it: any time — this is a low-risk cleanup, not a design change.*
5. **Make adding a new specialist a config change, not new code.**
   Right now, adding specialist #5 means writing a new Python function that
   follows the shared recipe by convention. A simple lookup table (agent
   name → prompt file → tool list → model tier) that the shared harness
   reads generically would make this closer to "just add a row."
   *Worth it: once you're adding a 3rd or 4th specialist beyond today's
   four — premature for a from-scratch build of just these four.*

None of these mean "this is broken." The current design is a solid,
low-complexity starting point — treat this list as ideas to revisit later,
if and when the matching pain point actually shows up.

---

## Build Checklist

A suggested build order for reproducing this system from scratch — each step
should work and be testable on its own before moving to the next one:

1. **Data models**: a conversation, a message (role/content/history), and —
   only if you want the async/polling delivery pattern — a durable "run"
   record (status, progress text/percent, ordered step trace, error
   message). Skip the run record for a first pass and just call the
   Orchestrator directly and wait for the answer.
2. **LLM provider wrapper**: a model-builder function per tier, each
   returning a primary and backup client, plus one shared
   `invoke_with_fallback` helper that retries once on any failure.
3. **Tool functions**: start with the General agent's calendar/time tools
   (simplest, nothing external needed), then the Tool agent's downtime/
   missing-report tools (these need your real data source).
4. **Shared sub-agent harness**: prompt-building (static template + dynamic
   context lines), tool-calling agent construction/caching, message-list
   building, and the uniform result shape (`{source, result, tools_used,
   raw_tool_outputs}`).
5. **General agent**: the simplest possible specialist — proves the harness
   works end-to-end before adding any complexity.
6. **Tool agent**: adds the "don't guess, ask" clarification pattern and a
   tool whose output format the agent must leave alone.
7. **Analytics agent**: adds the write-and-run-SQL loop first; add the
   knowledge-graph root-cause path and the tool-free summary mode after the
   basic SQL loop is working.
8. **Router**: build the keyword fast-path first (try it against a handful
   of real example queries), then add the LLM fallback for whatever the
   fast-path can't confidently handle.
9. **Orchestrator**: wire Router → invoke-with-fallback → (optionally)
   progress-event streaming. Get the plain, non-streaming `invoke()` path
   working end-to-end *before* adding the progress-event layer on top.
10. **Async execution + delivery** (optional, add once everything above
    works): move Orchestrator calls into a background worker, add the
    durable run record + a poll endpoint, add cancellation.
11. **Technician/document agent**: stub it as a placeholder route until its
    document-retrieval pipeline exists as its own project; wire it in for
    real once that's ready.

**The test at every step:** does this specialist (or router decision)
produce the right answer/route for a handful of realistic sample questions,
using conversation history where it matters? Don't move on until that's
true.
