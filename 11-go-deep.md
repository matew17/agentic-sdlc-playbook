# 11 — Go Deep / Explore More

> This is the "what next" file. Once the pilot is running and the habits stick, these are the four directions worth investing in — and the ones to avoid. Opinionated as ever.

Four topics:

1. [Memory — beyond markdown](#1-memory--beyond-markdown)
2. [Multi-agent frameworks — the honest case](#2-multi-agent-frameworks--the-honest-case)
3. [Evals — the highest-leverage investment you're not making yet](#3-evals--the-highest-leverage-investment-youre-not-making-yet)
4. [Observability-as-code — from run logs to queryable traces](#4-observability-as-code--from-run-logs-to-queryable-traces)

For each: what it is, when it starts earning its keep, what to read, what to avoid.

---

## 1. Memory — beyond markdown

You started with **file-based memory**: `CLAUDE.md`, `docs/adr/`, `docs/prds/`, `docs/agent-runs/`. This is correct for your pilot and will remain correct for most projects for a long time. Do **not** upgrade until you hit a real ceiling.

### When the markdown model hits its ceiling

- You can feel context-window pressure weekly (agent asks to re-read things it should remember).
- You have ≥ 5 repos that share architectural conventions and you keep duplicating `CLAUDE.md`.
- You want cross-repo search over decisions ("has any team solved X?").
- You want to feed historical incidents into planner context automatically.
- Compliance asks for a queryable audit surface beyond git.

Until you hit **at least two of these**, don't touch your memory stack.

### Upgrade path A — Curated retrieval (the sane first step)

Before reaching for a vector DB, add **deterministic retrieval**: a small script that, at agent start, pulls in the 3–5 most relevant docs based on the branch name, labels, or file paths touched. This is 80% of the value of RAG at 5% of the complexity.

Concrete: a prompt pre-processor that runs `grep -l -r "<component-name>" docs/adr/ docs/prds/` and prepends matches to the first agent message.

**Read:** Anthropic's context-engineering post. [Context is all you need](https://www.anthropic.com/engineering/effective-context) (or whichever the current canonical piece is — search Anthropic's engineering blog).

### Upgrade path B — Vector DB + embedding retrieval

Reach for this when curated retrieval stops scaling (hundreds of ADRs, multiple repos, fuzzy queries). Recommended stack if/when you get there:

- **Embedding model:** Voyage-3 or Cohere Embed v3. Use text-embedding-3-large from OpenAI only if you already have their infra.
- **Store:** pgvector (if you already run Postgres), Turbopuffer (serverless + cheap at scale), or Pinecone (managed).
- **Indexing:** embed on commit via a GitHub Action. Source of truth stays in git; the vector DB is a cache.
- **Retrieval interface:** an MCP server that the agent calls like any other tool. See [MCP docs](https://docs.claude.com/en/docs/claude-code/mcp).

Avoid:
- **Building retrieval into the agent loop directly.** Keep it as an MCP tool call; it's cleaner and swappable.
- **Syncing the vector DB from anywhere but git.** Let git be the source of truth; re-embed on push.
- **LangChain's memory abstractions.** Overkill; opaque; easier to write 50 lines yourself.

### Upgrade path C — Managed agent memory (Mem0, Zep, similar)

These products provide "long-term memory" — episodic + semantic — with opinionated APIs. They shine in **conversational agents** (customer support, personal assistants) where user-specific memory matters. For **SDLC work**, they're a mismatch — you want repo-scoped, auditable, diff-able memory. Git does this better.

Revisit only if your client builds an end-user-facing AI feature that remembers across sessions.

### Upgrade path D — Knowledge graph

Extract entities and relations (components, decisions, owners, incidents) into a graph (Neo4j, Kùzu). Query with Cypher or similar. Useful when questions become relational ("which services depend on X and had an incident last quarter?").

Expensive to build, pays off in large orgs with many repos. **Do not** build this at the pilot stage.

### Further reading

- [Claude Code MCP](https://docs.claude.com/en/docs/claude-code/mcp)
- [Model Context Protocol spec](https://modelcontextprotocol.io/)
- [Chroma / pgvector tutorials](https://github.com/pgvector/pgvector) for hands-on
- "Building effective agents" from Anthropic Engineering — reread the "tools vs. memory" section when you're ready to upgrade.

### Quick decision rule

| Signal | Action |
|---|---|
| You maintain CLAUDE.md by hand per repo | Markdown is fine. Keep. |
| You copy CLAUDE.md between repos | Extract shared bits into a shared git repo, load via submodule or a download step. |
| You search across ADRs weekly | Curated retrieval (grep-based). |
| Agent asks for the same doc repeatedly | Retrieval via MCP tool. |
| Compliance asks for cross-repo decision search | Vector DB + MCP. |
| Your org has 50+ repos and needs lineage | Knowledge graph. |

---

## 2. Multi-agent frameworks — the honest case

You skipped CrewAI / AutoGen / LangGraph this week. Good. For SDLC work with structured handoffs, **Claude Code subagents + orchestrator is the right shape.**

When would you actually reach for a framework?

### Legitimate reasons

1. **Compliance forbids a CLI runtime** in your CI. You need a server-side agent harness.
2. **Your tasks are genuinely unstructured** (research, content, customer-support triage) and the workflow shape changes run-to-run.
3. **You need tight integration with non-Anthropic LLMs** and want provider-agnostic routing.
4. **You want a visual workflow editor** for non-technical users to compose agents.

### Honest comparison (circa 2026)

| Tool | Shape | Best at | Watch out for |
|---|---|---|---|
| **LangGraph** | Graph-of-nodes state machine | Explicit control flow; complex routing | Verbosity; debugging is painful; LangChain baggage |
| **CrewAI** | Role-based agent crews | Quick demos, opinionated roles | Non-determinism; message loops; production hardening |
| **AutoGen (Microsoft)** | Group-chat multi-agent | Research + exploration | Same as above; moving target |
| **Pydantic AI / Instructor** | Typed agent primitives | Clean data validation | Minimal — these are libraries, not frameworks |
| **OpenAI Agents SDK** | Single-vendor agent primitives | If you're OpenAI-centric | Lock-in; less control |
| **Anthropic SDK (raw)** | Build-your-own | Minimal surface; total control | You write the orchestration (~200 LOC for SDLC) |

### My opinion

If you outgrow Claude Code for SDLC orchestration (you probably won't, this year), skip the frameworks and **write it yourself against the Anthropic SDK**. A router + five functions (planner, implementer, tester, reviewer, security) is 200 lines of TypeScript or Python. You'll understand every token, every failure mode, every cost.

The moment you add a framework, debugging shifts from "my prompt is wrong" to "is this a framework bug or my bug?" — not a trade you want to make for the handful of lines you'd save.

### Further reading

- [Anthropic — Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) — section on "workflows vs agents."
- [LangGraph docs](https://langchain-ai.github.io/langgraph/) — skim only, for vocabulary.
- [OpenAI Agents SDK](https://platform.openai.com/docs/guides/agents) — if your org pivots to OpenAI.

### Exit criterion (when to revisit)

You tried to build a workflow your orchestrator can't express cleanly, AND you'd describe the task as "open-ended reasoning with dynamic branching," AND you've lived with the limitation for 2+ weeks. Then — and only then — explore frameworks.

---

## 3. Evals — the highest-leverage investment you're not making yet

This is the single most under-invested area in agentic SDLC work, and the one that separates teams who *think* their agents are good from teams who *know*.

### What an eval suite looks like for agentic SDLC

A set of **frozen tasks** you can re-run against any version of your agent system, each with:
- A deterministic input (task brief, repo state via git sha).
- A clear expected outcome (passing tests + specific assertions on diff shape).
- An automated scorer (usually: tests pass + the agent-generated code conforms to rules you can check programmatically).

Build ~10 such tasks — a mix of easy (rename a file across the codebase), medium (add a feature with tests), hard (refactor a composable with backward compatibility). Run them weekly against your current prompt/subagent config.

Now when you consider: "should I switch planner from Haiku to Sonnet?" — you have numbers, not vibes.

### Minimal eval setup

```
evals/
├── tasks/
│   ├── 01-rename-prop/
│   │   ├── task.md         # input
│   │   ├── before/         # git sha or snapshot
│   │   ├── expected.md     # success criteria
│   │   └── score.sh        # returns 0 on pass
│   ├── 02-add-search-field/
│   └── ...
├── run-eval.sh             # loops tasks, captures cost + time + pass/fail
└── report/
    └── YYYY-MM-DD.md       # latest run
```

Run in CI on every change to `.claude/agents/*`, `.claude/skills/*`, or `CLAUDE.md`. Block the merge if pass rate drops.

### Further reading

- [Anthropic — Building an evals-first workflow](https://docs.claude.com/en/docs/test-and-evaluate/eval-tool) (or current canonical link).
- [Braintrust](https://braintrust.dev/) and [Humanloop](https://humanloop.com/) — managed eval platforms; useful if you want dashboards without building them.
- [Inspect AI](https://inspect.ai-safety-institute.org.uk/) — rigorous eval framework from the UK AISI.

### When to invest

**Week 3 at the client, latest.** Even a 5-task suite beats none. This is also the artifact most likely to convince skeptical leadership — "we can measure the agent."

---

## 4. Observability-as-code — from run logs to queryable traces

Run logs + PR descriptions are the right starting point and they may be all you need for a year. When they stop being enough:

### Symptoms that it's time

- You have questions like "how often does the implementer hallucinate a file path?" and cannot answer without reading 50 run logs.
- You want per-run cost attributed to model tier, subagent, and task type.
- Leadership asks for weekly KPIs and you don't want to compile them by hand.
- You suspect a regression in agent quality but can't prove it.

### Three tiers of upgrade

**Tier 1 — Structured run logs.** Change `docs/agent-runs/<task>.md` from prose to JSON frontmatter + prose body. Parse with a weekly script. You get queryable metrics for free; your audit trail stays human-readable.

Example frontmatter delta:
```yaml
---
subagent_calls:
  - agent: planner
    model: haiku
    duration_s: 42
    tokens_in: 8200
    tokens_out: 1400
    result: success
  - agent: implementer
    model: sonnet
    duration_s: 180
    tokens_in: 24000
    tokens_out: 2800
    result: success
outcome: merged
incidents: []
---
```

**Tier 2 — OpenTelemetry traces via the Anthropic SDK.** When you move agent runs out of Claude Code into a custom harness, emit OTel spans per subagent call. Ship to Honeycomb, Datadog, or Grafana Tempo. You now have distributed tracing for agent workflows.

**Tier 3 — Purpose-built tools.** [Langfuse](https://langfuse.com/), [Arize Phoenix](https://phoenix.arize.com/), [Helicone](https://www.helicone.ai/) — LLM-native observability platforms. Useful if you go multi-vendor or run 100+ agent invocations/day.

### What to avoid

- Building a custom dashboard from scratch before knowing what you'd measure. You'll build the wrong one.
- Adding observability to Claude Code itself. Its internal telemetry is what it is; measure at the boundary (the wrapper workflow or the CI job).
- Treating observability as a week-1 task. It's a month-2+ investment, paid for by real pain you've actually felt.

### Further reading

- [OpenTelemetry for LLMs](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — semantic conventions.
- [Langfuse self-host guide](https://langfuse.com/docs/deployment/self-host) if compliance requires on-prem.

---

## What I am deliberately leaving out of this section

- **Fine-tuning Claude.** Not available, not the right lever anyway. Better prompts, better subagents, better skills — in that order.
- **Running local LLMs for SDLC work.** In 2026, the gap from frontier is still too wide for production SDLC autonomy. Revisit when OSS closes it.
- **Agent marketplaces / plugin ecosystems.** Interesting; not load-bearing. Don't build your architecture around one.
- **RLHF / DPO on agent outputs.** Research-grade. Not your problem this year.
- **Browser-using agents for SDLC.** Niche; your pipeline runs better without a browser. Keep browser agents for tasks that truly need them (visual QA, scraping).

---

## The underlying rule

Every upgrade in this file costs **complexity, debuggability, and team onboarding time**. Defaults have enormous gravitational pull — markdown memory, Claude Code subagents, run logs in git, human gates at three points — because they are legible and repairable by any senior engineer.

Reach for heavier tools only when a specific pain, experienced for at least two weeks, makes the simpler tool indefensible.

Back to the [top-level index](./README.md).
