# 02 — Target Architecture for Your Case

> Recommendation first. Reasoning second. Tradeoffs third.

---

## The recommendation: Orchestrator + specialized subagents, gated by git

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           HUMAN GATES (3)                                │
│   G1: Spec approval    G2: Pre-merge review    G3: Production release    │
└─────────────────────────────────────────────────────────────────────────┘
         ▲                        ▲                          ▲
         │                        │                          │
┌────────┴──────────┐    ┌────────┴──────────┐    ┌──────────┴────────┐
│  PLANNING PHASE   │ →  │ IMPLEMENTATION    │ →  │  RELEASE PHASE    │
│  (local / PR desc)│    │ (feature branch)  │    │ (main + deploy)   │
└───────────────────┘    └───────────────────┘    └───────────────────┘
         │                        │
         ▼                        ▼
┌───────────────────┐    ┌──────────────────────────────────────────────┐
│ CLAUDE CODE       │    │ CLAUDE CODE (local or GH Action)             │
│ + planner agent   │    │                                              │
│ + /prd command    │    │   orchestrator ──► implementer subagent      │
│                   │    │                ──► tester subagent            │
│ Output: PRD.md    │    │                ──► reviewer subagent          │
│         TASKS.md  │    │                ──► security subagent          │
└───────────────────┘    │                                              │
                         │   Tools: Read, Write, Edit, Bash (sandboxed), │
                         │          git, pnpm, vitest, playwright,       │
                         │          eslint, tsc, gh CLI                  │
                         │                                              │
                         │   Ground truth: tsc + vitest + playwright + eslint │
                         │                                              │
                         │   Output: PR with diff, tests, run log        │
                         └──────────────────────────────────────────────┘
                                         │
                                         ▼
                         ┌──────────────────────────────────────────────┐
                         │ GITHUB ACTIONS (agentic + deterministic)     │
                         │  - CI: build, typecheck, test, e2e, lint     │
                         │  - On failure: triage subagent posts summary │
                         │  - Required checks gate merge                │
                         └──────────────────────────────────────────────┘
```

### The five components

1. **Orchestrator (Claude Code main session).** Holds the task, owns the branch, decides which subagent to call. Never writes production code directly when a specialist exists.
2. **Planner subagent.** Turns a one-line request into a PRD, a task list, and a test strategy. Cheapest to re-run; iterate until the plan is right before any code is written.
3. **Implementer subagent.** Writes application code on a feature branch. Has `Edit`, `Write`, `Bash` (scoped), `Read` tools. Cannot push to `main`, cannot merge.
4. **Tester subagent.** Writes unit + e2e tests against the spec, not the implementation. Run separately from implementer to avoid "tests that match the buggy code."
5. **Reviewer subagent.** Reads the diff cold, runs the checklist, posts comments to the PR. Tuned for *skepticism*; its job is to find what the implementer missed.
6. *(Optional, day 6)* **Security subagent.** Runs on every PR via GH Action. Checks for secrets, auth mistakes, injection patterns, dependency CVEs.

The **orchestrator is always Claude Code**, not a custom Python harness. The specialists are Claude Code **subagents** (files in `.claude/agents/`), each with its own system prompt, tools, and model tier if needed.

---

## Why this architecture for you

You are optimizing for **four things at once**, and this shape is the only one that delivers all four:

| Goal | How this architecture delivers |
|---|---|
| **High autonomy** | Branches are sandboxes. Inside a branch, the agent is fully autonomous — plan, code, test, self-review. |
| **Safety** | Human gates are at the *contract boundaries* (spec, merge, release), not sprinkled throughout. CI enforces ground truth. |
| **Auditability** | Everything is a git artifact: branches, commits, PRs, run logs in `docs/agent-runs/`. A post-mortem is `git log`. |
| **Portability** | Subagents are markdown files. CLAUDE.md is markdown. Skills are markdown. This moves to any Claude-capable tool, and the *shape* (planner/implementer/tester/reviewer) moves to Devin trivially. |

This is also the shape that **Anthropic itself recommends** in "Building effective agents": prefer workflows with clear handoffs over open-ended agent loops whenever the task has structure. SDLC tasks *always* have structure.

---

## The authority model (memorize this)

| Actor | Read | Edit files | Run tests | git commit | git push | Open PR | Merge | Deploy |
|---|---|---|---|---|---|---|---|---|
| Orchestrator | ✅ | via subagents | ✅ | ✅ | ✅ (feature branch only) | ✅ | ❌ | ❌ |
| Planner | ✅ | only `docs/` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Implementer | ✅ | ✅ | ✅ | ✅ | ❌ (orchestrator pushes) | ❌ | ❌ | ❌ |
| Tester | ✅ | tests dir only | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Reviewer | ✅ | ❌ (comments only) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Security | ✅ | ❌ | scanners only | ❌ | ❌ | ❌ | ❌ | ❌ |
| Human | all | all | all | all | all | all | ✅ | ✅ |

**Enforcement mechanism:** Claude Code permissions config (`.claude/settings.json`), subagent `allowedTools` field, branch protection rules on `main`, required GH Actions checks, deploy step gated by `workflow_dispatch` + environment approval.

---

## Memory model (opinionated: keep it boring)

- **Durable project memory:** `CLAUDE.md` at repo root, plus `CLAUDE.md` files in subdirectories for local context. This is Claude Code's native memory and survives across sessions.
- **Decision memory:** `docs/adr/NNNN-title.md` — Architecture Decision Records. The planner writes these, humans review them.
- **Run memory:** `docs/agent-runs/YYYY-MM-DD-task-slug.md` — what ran, what it decided, what it produced, what needs follow-up.
- **Skill memory:** `.claude/skills/<name>/SKILL.md` — reusable procedures with referenced scripts/templates.
- **Shared human↔agent memory:** PR descriptions. Generated from the run log, readable by humans, re-ingestible by agents.

**No vector DB, no external memory service.** File-based memory is auditable, diff-able, portable, and sufficient for SDLC workloads. Revisit only if you hit real context pressure (you won't, this week).

---

## Control flow (the happy path)

```
1. Human creates GitHub issue: "Add favorites list to anime catalog"
2. Human runs: claude → /start-feature issue-42
3. Orchestrator calls planner → produces docs/prds/42-favorites.md + task list
   ◆ G1 (human approval of PRD)  ← push back here until right
4. Orchestrator creates branch feat/42-favorites
5. Orchestrator loops: implementer → tester → reviewer until reviewer is happy
6. Orchestrator runs local CI (tsc, lint, vitest, playwright); stops on red
7. Orchestrator opens PR with generated description from run log
8. GH Action re-runs CI and security subagent in clean env; results posted to PR
   ◆ G2 (human merges)  ← blocks until green + human ok
9. On merge to main: deploy workflow queued
   ◆ G3 (human approves prod deploy in GH Environments)
10. Post-deploy smoke subagent runs curl-based checks; posts status to PR
```

**Every arrow is logged.** Every subagent call appends to `docs/agent-runs/<task>.md`.

---

## What you gain vs. what you give up

### Gains
- **Parallelism-by-separation.** Tester doesn't see the implementer's reasoning, which kills "tests that match the bug."
- **Scoped blast radius.** Each subagent has narrow tools; one can't do catastrophic damage.
- **Upgradable parts.** Swap planner model, change reviewer checklist, add security subagent without re-architecting.
- **Clear narrative on PRs.** Run log → PR description → reviewers understand *why*.

### Costs
- **More moving parts** than a single-agent flow; you'll need a day to get the handoffs right.
- **Token spend is higher** — multiple calls per task. Mitigate with scoped context (per-subagent CLAUDE.md), Haiku for cheap passes, Opus/Sonnet only where it counts.
- **Orchestration bugs feel like a haunted house** on day one. Invest in run logs early and it gets easy.

### When to evolve away from this
- **Toward single-agent + tools:** if tasks become so small (one-line fixes, dep bumps) that the orchestration overhead dominates. See [03-architecture-alternatives.md](./03-architecture-alternatives.md).
- **Toward event-driven CI agents:** once the pilot is stable, migrate routine work (triage, dep updates, lint fixes, changelog) to GH-Action-triggered agents that run without human initiation.
- **Toward richer multi-agent** (planner-of-planners, critic-of-critic): only if you hit quality ceilings this shape can't clear. Most teams never need this.

Next: [03-architecture-alternatives.md](./03-architecture-alternatives.md).
