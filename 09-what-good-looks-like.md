# 09 — What Good Looks Like (your end-of-week capabilities)

> The exam. If you can honestly check every box, you're ready to lead this at the client.

---

## Capability 1 — Design

You can, on a whiteboard, in 20 minutes:

- Sketch an orchestrator + subagents diagram for a *new* project.
- Name each subagent's role, inputs, outputs, tools, boundaries, success criteria, and escalation path.
- Mark the three human gates and explain why they're exactly three.
- Identify at least three cautious paths for the target domain and justify each.

---

## Capability 2 — Build

On a clean repo:

- You can configure Claude Code with `CLAUDE.md`, `.claude/settings.json`, at least three subagents, and two hooks — in under 90 minutes.
- You can ship a feature end-to-end through the full pipeline with ≤ 45 minutes of human time, including PRD review.
- You can wire a PR-review-on-every-PR GitHub Action without copying a template you don't understand.

---

## Capability 3 — Operate

- You have a run log for every merged PR and can explain any past agent decision in ≤ 2 minutes by opening `docs/agent-runs/`.
- You have a written intervention policy and you (and your team) actually follow it.
- You can tell me, *right now*, what would happen if the agent tried to push to `main` tonight. It would fail, and you can name why: branch protection + deny list + no direct push capability.
- You have practiced firing the three kill switches.

---

## Capability 4 — Diagnose

Given a bad PR, you can:

- Classify the failure as Drift / Hallucination / Overreach / Silent wrongness / Over-reliance.
- Identify which artifact change (CLAUDE.md rule, subagent prompt, skill, hook, CI check) would have caught it.
- Apply the fix *to the system*, not just patch the PR.

---

## Capability 5 — Judge

You have clear, pre-committed answers to these questions and can defend them:

- "Should the agent merge its own PRs?" → No, never. Humans at G2.
- "Should the agent open a DB migration PR?" → Yes, but cautious mode; no auto-run in CI on production.
- "Should the agent deploy to production?" → No. Humans at G3 via GH Environments.
- "Should the agent review its own code?" → Separate reviewer subagent, yes. Self-review by the implementer, no.
- "Should the agent write tests *and* the code?" → Separate subagents, tests from spec, not from code.
- "Should we add a second agent framework for research tasks?" → Not for SDLC. Revisit if and only if task structure breaks down.

---

## Capability 6 — Communicate

You can:

- Walk a skeptical director through the architecture in 15 minutes using `02-target-architecture.md`.
- Show an auditor a complete trail for any change in ≤ 5 minutes: issue → PRD → ADR (if any) → branch → commits → PR → reviewer comment → CI → merge → deploy.
- Justify why you chose *this* architecture over swarms, single-agent, or human-only, without hand-waving.

---

## Capability 7 — Train others

- A teammate can clone the repo, read `README.md` and `CLAUDE.md`, and ship their first agent-driven PR solo within their first day.
- Your runbook on intervention (`docs/agentic-intervention-policy.md`) gives a non-expert a defensible default.
- You have a 30-minute onboarding talk outline (use the headings of `02`, `05`, and this file).

---

## Capability 8 — Know the limits

You can name five things you will *not* let the agent do this quarter, and why. Examples, pick or adapt:

1. Merge to main without a human.
2. Write code in `auth/`, `payments/`, `crypto/`.
3. Approve its own PRs.
4. Run DB migrations.
5. Deploy to production.

And five things you *will* let the agent do fully autonomously:

1. Dep patch bumps (with tests green).
2. Changelog updates.
3. Stale-issue hygiene.
4. PR review drafting.
5. CI failure triage commenting.

---

## Capability 9 — Measure

You can answer:

- How many PRs this week were agent-authored end-to-end? (Counts.)
- How many of those required material human changes before merge? (Quality.)
- How long from issue-open to PR-open? (Cycle time.)
- How many agent-authored PRs caused incidents or rollbacks? (Safety.)
- What did agent work cost this week in API spend? (Efficiency.)

Track these in a weekly 10-line note committed to `docs/metrics/week-NN.md`. Don't build a dashboard yet.

---

## Capability 10 — Restraint

This is the hardest, and the one that separates a principal from a mid-level operator:

- You can say "the agent is not the right tool for this" without embarrassment.
- You do not chase the latest framework; you evolve the current one only when a real failure forces it.
- You resist the urge to add a fifth subagent, a vector DB, a multi-level critic — until a failure you cannot solve otherwise demands it.
- You add process only when you have a specific failure to prevent, and remove it when it stops earning its keep.

---

## Week-after priorities (what to do on day 8 and beyond)

Now that the pilot is live, the next two weeks at the client look like:

- **Week 1 at client:** deploy reviewer-only + planner-only mode. Earn trust. Baseline the metrics above.
- **Week 2:** enable implementer on non-critical paths. Keep cautious paths fully human.
- **Week 3+:** expand scope by scoring each path on the decision matrix. Retrospectives every Friday using `09` as the exam.

---

## Closing thought

You're not building "an AI that writes code." You're building an **operating model** where a bounded, observable, auditable agent participates in your SDLC with the same professional expectations as any engineer — documented scope, reviewed output, passing tests, traceable decisions — and where the only thing different is *where the labor goes*.

Humans do more **deciding**. Agents do more **producing**. Both do less **ceremony**.

If your week-1 pilot demonstrates that balance once, you have enough to scale it to the client.

Good luck. Now go ship.

— End of playbook —

Back to [README.md](./README.md).
