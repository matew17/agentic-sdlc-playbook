# 04 — The 7-Day Roadmap

> 21 hours total. Each day is a self-contained file with explicit per-step instructions and the exact artifacts to use.

## At a glance

| Day | Theme | End-of-day artifact | File |
|---|---|---|---|
| 1 | Foundations + repo baseline | Vue repo has CI, lint, typecheck, one e2e scaffold, `CLAUDE.md` v1 | [day-1.md](./day-1.md) |
| 2 | Single-agent primitives in Claude Code | First full task run end-to-end (plan → code → test → PR) | [day-2.md](./day-2.md) |
| 3 | Subagents + skills | Planner, implementer, tester, reviewer + one skill | [day-3.md](./day-3.md) |
| 4 | Hooks, permissions, human gates | Authority model enforced; cautious mode for sensitive paths | [day-4.md](./day-4.md) |
| 5 | CI-triggered agents + GitHub Actions | Agent reviews every PR; triage subagent on failed CI | [day-5.md](./day-5.md) |
| 6 | Production hardening + observability | Run logs, ADRs, secret scanning, cost discipline | [day-6.md](./day-6.md) |
| 7 | Pilot feature end-to-end + playbook polish | One real feature shipped agentically + reusable playbook | [day-7.md](./day-7.md) |

## How each day is structured

Each file follows the same shape so you can navigate without thinking:

- **Objective** — one line.
- **Concept primer** — why this day matters; no bloat.
- **Prerequisites** — what from earlier days must be in place.
- **Time plan** — step/time breakdown (fits 3 hours).
- **Steps** — each step has: **Do**, **Artifact for this step** (link into [`06-artifacts.md`](../06-artifacts.md)), **Check**.
- **Checkpoints** — how to know you're still on track.
- **Definition of Done** — the merge bar for the day.
- **Cut-line** — what to drop if time runs short.
- **Mistakes to avoid**, **validation signals**, **references**.
- **Today's artifact bundle** — the minimal list of what you committed.

## Weekly retro (Sunday, 20 minutes)

Write `docs/agent-runs/week-1-retro.md` answering:

1. What decisions would I redo?
2. Which subagent earned its keep? Which didn't?
3. Where did I intervene and shouldn't have? (Too cautious.)
4. Where did I not intervene and should have? (Too autonomous.)
5. What will I change before the client project starts?

Back to the [top-level index](../README.md).
