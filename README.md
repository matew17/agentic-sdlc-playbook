# Agentic SDLC Playbook — Claude Code Edition

> One week. Three hours a day. One working pilot. One reusable framework you can carry to your HR/Fintech client.

This is a roadmap + playbook + reusable framework for a senior engineer who already knows architecture, testing, and CI/CD, and wants to learn to **design, operate, and integrate autonomous agentic systems into a real SDLC** — with Claude Code as the primary runtime.

---

## Executive Summary

**The core thesis.** The future of SDLC automation is not "AI writes code for you." It is a **fleet of narrow, auditable agents** orchestrated inside your existing engineering contract — branches, PRs, CI, code review, release gates. The winning architecture is **composable**: one primary agent (Claude Code) that delegates to specialized subagents, with deterministic tools (tests, linters, builds, git) as the ground truth, and **humans at exactly three gates**: spec approval, pre-merge review, and production release.

**What you will build.** By day 7 you will have a Vue 3 + Vite + TypeScript anime-catalog repo that can, with a human pressing "approve" three times:

1. Take a one-line feature request and produce a PRD + task plan.
2. Decompose it into a branch, implementation, unit + e2e tests, and a self-reviewed PR.
3. Run CI with agent-assisted triage on failures.
4. Emit auditable logs for every agent decision.

**What you will carry to the client.** A stack-agnostic playbook with decision matrices, CLAUDE.md patterns, subagent definitions, skills, `/commands`, hooks, and a GitHub Actions starter. You will know **when to let the agent run and when to pull the brake** — that is the actual deliverable.

**Opinionated top-line calls** (each defended later):

| Decision | Recommendation |
|---|---|
| Architecture | **Orchestrator + specialized subagents** (planner / implementer / tester / reviewer), *not* a single mega-agent, *not* a CrewAI-style swarm. |
| Runtime | **Claude Code** locally + **Claude Code Action (or equivalent GH Action)** in CI for async tasks. |
| Memory | **File-based, versioned.** `CLAUDE.md`, `.claude/`, `docs/adr/`. No vector DB yet. |
| Control plane | **Git + GitHub PRs** — branches are your "agent runs," PRs are your "diffs to approve," CI is your ground truth. |
| Human gates | **3**: spec sign-off, pre-merge review, prod deploy. Everything else autonomous. |
| Autonomy dial | **High inside a sandboxed branch. Zero across the merge boundary.** |
| Observability | Structured markdown run logs committed to `docs/agent-runs/` + PR descriptions generated from them. |

---

## How to use this playbook

Read in this order. Each file is self-contained, but they build on each other.

| # | File | Purpose |
|---|---|---|
| 1 | [01-learning-strategy.md](./01-learning-strategy.md) | How to actually spend your 21 hours. Study strategy and anti-patterns. |
| 2 | [02-target-architecture.md](./02-target-architecture.md) | The recommended agentic architecture for your scenario, with reasoning. |
| 3 | [03-architecture-alternatives.md](./03-architecture-alternatives.md) | Alternatives compared honestly. When to evolve away from the recommendation. |
| 4 | [04-roadmap/](./04-roadmap/README.md) | The 7-day execution plan. One file per day; each step paired with the exact artifact to use. |
| 5 | [05-playbook.md](./05-playbook.md) | Reusable framework you apply to future projects. Decision flows and rituals. |
| 6 | [06-artifacts.md](./06-artifacts.md) | Canonical reference: prompts, subagents, skills, templates, checklists, CI, repo layout. Daily files link back here. |
| 7 | [07-failure-modes.md](./07-failure-modes.md) | How agentic systems fail in production and how to prevent each mode. |
| 8 | [08-tool-transfer.md](./08-tool-transfer.md) | Mapping Claude Code concepts to Devin and other tools. |
| 9 | [09-what-good-looks-like.md](./09-what-good-looks-like.md) | End-state capabilities you should have on day 8. |
| 10 | [10-task-sources.md](./10-task-sources.md) | Swap GitHub Issues for Linear / Jira / Asana without touching the pipeline. Worked Linear example. |
| 11 | [11-go-deep.md](./11-go-deep.md) | What to explore next: memory upgrades, multi-agent frameworks, evals, observability-as-code. |

---

## Ground rules for this week

1. **Ship over study.** If a concept does not turn into a file, a prompt, or a PR today, cut it.
2. **Anchor on the anime project, generalize in the playbook.** Never let Vue-specific code leak into the reusable framework.
3. **No demo-grade shortcuts.** If something would not survive a production post-mortem, flag it as tech debt in `docs/agent-runs/`.
4. **Read the diff.** Every autonomous agent output gets reviewed as if a junior engineer wrote it. That is the whole job.
5. **One week, one pilot, one playbook.** Not three.

Now go to [01-learning-strategy.md](./01-learning-strategy.md).

### A reading order for different moods
- **First read through** — go in order 01 → 11.
- **Starting the week** — skim `01`, `02`, then live in `04-roadmap/` day by day.
- **Setting up at the client** — start from `05-playbook.md` + `10-task-sources.md`.
- **Something's broken** — jump to `07-failure-modes.md`.
- **Planning the next quarter** — `11-go-deep.md`.
