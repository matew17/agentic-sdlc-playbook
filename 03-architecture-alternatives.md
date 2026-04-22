# 03 — Architecture Alternatives & Tradeoffs

> What I did not recommend, and why. Use this when the client pushes back or when conditions change.

---

## Alternative A — Single-agent + tools

**Shape.** One Claude Code session, no subagents, broad tool access. Agent does everything.

**When it wins.** Tiny repos. Solo hobby projects. One-shot scripts. Experiments. "Fix this lint rule across the repo."

**When it loses.**
- No separation between writer and critic → weak reviews.
- Tests written by the same reasoning that wrote the code → tests encode the bug.
- One giant context window → drift, hallucination, scope bloat.
- Hard to upgrade parts independently.

**Verdict for your case.** ❌ Underpowered. Keep it in your toolbox for dev-loop micro-tasks (one-file refactor, rename, codemod). Don't build your pilot on it.

---

## Alternative B — Full multi-agent swarm (CrewAI / AutoGen / LangGraph style)

**Shape.** Dedicated agent framework. N agents with role cards negotiate via messages. Custom Python harness.

**When it wins.** Research problems. Open-ended tasks where the solution shape is unknown. Non-SDLC workflows (marketing content, customer support triage).

**When it loses.**
- **Non-determinism compounds.** Three agents chatting produces reasoning loops and token waste.
- **You re-implement what Claude Code gives you for free** — file tools, bash, git, permissions, context management.
- **Portability is worse**, not better. The framework becomes the lock-in.
- **Debuggability is terrible.** Try reading a LangGraph trace at 2 AM.

**Verdict for your case.** ❌ Not this week, possibly never for SDLC. Claude Code's subagent model achieves the same role separation with orders-of-magnitude less scaffolding. Note the exception: if your client's security team forbids a CLI runtime in CI, a headless harness with the Anthropic SDK may be required — but keep the *shape* (orchestrator + specialists) identical.

---

## Alternative C — Event-driven CI-only agents

**Shape.** No local agent loop. Agents live entirely in GitHub Actions triggered by events (issue opened, PR opened, schedule, label added).

**When it wins.** Routine maintenance that doesn't need human conversation — dep updates, flaky-test triage, changelog generation, stale-issue grooming, security patching, release notes.

**When it loses.**
- **Feature development feels async and slow.** You want to iterate on a PRD conversationally, not by editing an issue.
- **Hard to interrupt and redirect mid-run.**
- **Harder to develop new workflows** — iteration cycle is "edit YAML, push, wait for workflow."

**Verdict for your case.** ✅ *As a complement.* Routine work belongs in CI. Feature work belongs in a local orchestrator. **Your target architecture uses both** — this is not an either/or.

---

## Alternative D — Human-in-the-loop everywhere

**Shape.** Agent pauses for human confirmation at every step. "Should I run tests?" "Should I commit?" "Should I write this line?"

**When it wins.** High-stakes code with low reversibility: infra-as-code for prod, migrations, security-critical code, regulated workloads (yes, fintech).

**When it loses.**
- **Autonomy collapses to zero.** You've built an autocomplete, not an agent.
- **Humans stop reading** after the 20th dialog and rubber-stamp.

**Verdict for your case.** ❌ as a default, ✅ as a mode. Define it as a **"cautious mode"** you switch into for specific paths (e.g., anything touching `payments/`, `auth/`, `db/migrations/`). In Claude Code, implement via hooks that block edits in those paths without explicit confirmation.

---

## Alternative E — Planner-only with human coder

**Shape.** Agent does PRD, planning, test strategy, PR review. Humans write the code.

**When it wins.** High-seniority teams who trust their own code more than an agent's, in domains where correctness is load-bearing (again: fintech).

**When it loses.** You leave most of the productivity on the table. And your code reviews — the hardest part — are still human bottlenecks.

**Verdict for your case.** A sensible **starting posture for the client** for the first 2–3 weeks. Ship planner + reviewer + security subagents first; earn trust; then enable the implementer subagent on non-critical paths. **For your learning pilot this week, still build the full stack** so you understand the whole chain.

---

## Alternative F — Reviewer-only (agent as code review bot)

**Shape.** Agent never writes code. It reviews every PR, comments, suggests changes, scans for security issues.

**When it wins.** Extremely conservative orgs. Day 1 of adoption. When you want zero risk.

**When it loses.** Limited upside. Teams quickly realize it's "Copilot for PRs" and want more.

**Verdict for your case.** Use as **week-1 mode at the client** to build organizational trust. Then expand.

---

## Decision table — which mode to run when

| Context | Architecture |
|---|---|
| Your pilot, this week | **Orchestrator + specialists** (the recommendation in `02`). |
| Client week 1–2 | **Reviewer-only + Planner-only** to earn trust. |
| Client week 3+ | **Full recommendation on non-critical paths.** |
| Paths under `auth/`, `payments/`, `db/migrations/` | **Cautious mode** (Alternative D) — human approves every edit. |
| Dep updates, changelog, triage | **Event-driven CI agent** (Alternative C). |
| Quick refactor in your own repo | **Single-agent + tools** (Alternative A) is fine. |
| Research spike on agent approaches | **Swarm framework** (Alternative B) only if truly unstructured. |

---

## The one graph to keep in your head

```
            ┌────────────────────────────────────────────┐
            │             Task structure                  │
            │   structured ←─────────────────→ open-ended │
            └────────────────────────────────────────────┘
                   │                            │
                   ▼                            ▼
         Workflow with handoffs          Agent loop
         (your SDLC work)                (research, content)
                   │
                   ▼
         Orchestrator + specialists ← you are here
```

SDLC work is **structured**. That is your single most important architectural fact. Do not let frameworks with agent-to-agent chatter sneak into your design; you'd be introducing open-ended-ness where determinism is available.

Next: [04-roadmap.md](./04-roadmap.md).
