# 01 — Learning Strategy for Your Exact Scenario

> You are a senior/principal-track engineer with 21 hours and a client deadline. Your gap is **agent design**, not tooling familiarity. Study accordingly.

---

## The only strategy that works for you

**Learn by shipping the pilot.** Every concept must either (a) become a file in the anime repo or (b) become a reusable artifact in the playbook. Reading ends the moment your hands know what to build next.

You are not learning "how Claude Code works." You are learning **how to design an SDLC where agents are first-class contributors with bounded authority**. Claude Code is the instrument; the craft is systems design.

### The 60/30/10 allocation

For every 3-hour block:

- **60% build** — configure Claude Code, write CLAUDE.md, author subagents/skills, wire hooks, run agent tasks, review diffs, push PRs.
- **30% design decisions** — write ADRs or decision notes. If you can't articulate *why* you chose something, you don't own it.
- **10% targeted reading** — only the specific docs/blog posts referenced at the end of each day. No generic "AI courses."

If you fall behind, cut the reading first, never the build.

---

## Mental model to carry all week

Hold these four axes in your head on every decision:

1. **Authority** — What is the agent allowed to do? (Read? Edit? Push? Merge? Deploy?)
2. **Ground truth** — What deterministically validates the agent's work? (Types, tests, lint, build, e2e, humans.)
3. **Checkpoint** — When does a human look at the output? (Never / PR / release / runtime alert.)
4. **Reversibility** — How do I undo this if it's wrong? (Revert commit / feature flag / DB rollback / redeploy.)

A well-designed agentic workflow has explicit answers to all four for every step. An under-designed one has vibes.

---

## What to aggressively cut

- **Prompt engineering theory** past what's in `06-artifacts.md`. You know how to write prompts.
- **LangChain / CrewAI / AutoGen tutorials.** They're a distraction this week. Claude Code + subagents covers your needs and maps cleanly to Devin later.
- **Vector DB / RAG setup.** You don't need it for SDLC work. File-based memory and git history are stronger and auditable.
- **Frontend-specific AI features** (in-product AI). Different problem.
- **Chain-of-thought prompt micro-optimization.** Claude Code handles reasoning internally; your job is scoping and context.
- **Anything that requires more than one afternoon to set up.** If it doesn't fit in a day, it's not in your pilot.

---

## What to double down on

- **Writing excellent `CLAUDE.md` files.** This is the highest-leverage artifact in Claude Code. Spend real time here.
- **Subagent design.** Narrow, composable, with clear contracts. Treat them like microservices.
- **Hooks and permissions.** The difference between a toy and a production system.
- **Git + CI as the contract.** Branch protection, required checks, PR templates — these are your safety rails.
- **Human gate policy.** Write it down, commit it, enforce it.
- **Run logs and ADRs.** If an agent run is not auditable, it did not happen.

---

## Study posture per day

- **Morning 10 minutes**: reread the day's DoD in `04-roadmap.md`. Visualize the artifacts you'll produce.
- **End of day 10 minutes**: write a 1-page "what I learned / what I'd redo" note in `docs/agent-runs/day-N.md`. This becomes part of your playbook.
- **Weekly delta**: Friday night, do a retro against [`09-what-good-looks-like.md`](./09-what-good-looks-like.md).

---

## Validation signals you're on track

- By end of **Day 2**, you can run Claude Code, point it at a ticket, get a PR, and articulate *exactly* which safety rails prevented which failure mode.
- By end of **Day 4**, you have two production-minded subagents and one skill that together pass a "would I let this run overnight on my client's repo?" smell test.
- By end of **Day 7**, a teammate could clone your repo, run one command, read two pages, and ship a feature via the agent pipeline.

If any of these is weak, you over-studied and under-shipped. Course-correct.

---

## References (only these — don't go wider)

- [Claude Code documentation](https://docs.claude.com/en/docs/claude-code/overview) — primary reference all week.
- [Subagents](https://docs.claude.com/en/docs/claude-code/sub-agents) — day 3.
- [Skills](https://docs.claude.com/en/docs/agents-and-tools/skills) — day 3.
- [Hooks](https://docs.claude.com/en/docs/claude-code/hooks) — day 4.
- [Claude Code GitHub Actions](https://docs.claude.com/en/docs/claude-code/github-actions) — day 5.
- [MCP](https://docs.claude.com/en/docs/claude-code/mcp) — day 6, lightly.
- [Anthropic: "Building effective agents"](https://www.anthropic.com/engineering/building-effective-agents) — read once on day 1. The rest of the week is applying it.

Next: [02-target-architecture.md](./02-target-architecture.md).
