# Day 2 — Single-Agent Primitives: One End-to-End Run

Nav: [Roadmap overview](./README.md) · [← Day 1](./day-1.md) · Next: [Day 3 →](./day-3.md)

## Objective
Ship one real feature through Claude Code's default single-agent flow. **You must feel the failure modes before you design around them.** This day is deliberately un-architected so the contrast with Day 3 is visceral.

## Concept primer (5 min)
A good single-agent run is a **loop**: understand → plan → edit → validate → commit. Ground truth (tests, tsc) lives in "validate." Without it, the loop doesn't converge. You'll watch this loop run once, note where it wobbles, and that becomes your brief for Day 3.

## Prerequisites
- Day 1 DoD met.
- `gh` CLI works for `gh pr create`.

## Time plan (3h)

| Step | Time | Outcome |
|---|---|---|
| 1 | 20 min | Write a small, scoped task brief |
| 2 | 60 min | Plan-mode run + review + approve |
| 3 | 45 min | Let Claude Code implement with supervision |
| 4 | 30 min | Add tests (unit + e2e) via the agent |
| 5 | 25 min | Open PR; write the run log by hand |

---

## Step 1 — Write the task brief (20 min)

### Do
Pick a small but non-trivial feature. Recommended: **"Add a search bar to the anime list that filters by title, case-insensitive, with 250ms debounce."**

Create `docs/tasks/001-search.md` using the task-brief pattern.

### Artifact for this step
- [Task brief pattern](../05-playbook.md#task-brief-pattern-prompt-template) in the playbook. Fill it in full — this is the exact prompt you will hand the agent.

### Check
- Your brief has explicit **Acceptance** bullets (functional + non-functional).
- **Out of scope** is explicit (e.g., "no API changes, no new routes").
- **Autonomy level** is set to `cautious` for today — you want to see the agent work, not rubber-stamp.

---

## Step 2 — Plan-mode run (60 min)

### Do
1. Start Claude Code, paste the task brief.
2. `Shift+Tab` to switch to **plan mode**.
3. Let it produce a plan. **Read every line.** Push back at least once — too vague, missing a non-functional, unclear file placement.
4. Approve when the plan is tight: 3–7 concrete steps, named files, explicit test intent.

### Artifact for this step
- No deliverable file yet. But **screenshot or copy the plan into `docs/agent-runs/day-2.md`** — tomorrow's planner subagent is modeled on what a good plan looks like.

### Check
- You pushed back at least once.
- The final plan mentions the test it will write, not just "add tests."

---

## Step 3 — Implementation with supervision (45 min)

### Do
1. Exit plan mode; let the agent apply edits one batch at a time.
2. **Read every diff.** This is the single highest-leverage habit of the week.
3. When the agent wants to expand scope (rename adjacent code, "refactor while we're here") — say **no**. Pin the scope.
4. Run `pnpm typecheck && pnpm lint` after each batch. Fix or let the agent fix.

### Artifact for this step
- None produced; you are *consuming* the effects. Capture two inline moments in your run log where the agent almost went off-rails.

### Check
- Diff is scoped to search-related files (component, composable, maybe a util).
- Typecheck + lint green.
- No new dependency unless justified (and noted).

---

## Step 4 — Tests via the agent (30 min)

### Do
Ask the agent to add:
1. A **unit test** for the debounce/filter composable (e.g., `useFilterAnime.spec.ts`).
2. One **e2e assertion** in `e2e/search.spec.ts`: type a query, assert filtered results.

Do **not** accept "tests pass" on faith. Read the test code and verify it actually exercises the new behavior.

### Artifact for this step
- Reference the [test strategy template](../06-artifacts.md#test-strategy-template) when asking the agent — insist it maps AC → test.

### Check
- `pnpm test && pnpm test:e2e` green.
- Each test clearly asserts behavior from the task brief.

---

## Step 5 — Open the PR and write the run log by hand (25 min)

### Do
1. Commit in small narrative commits (`feat(search): add filter composable`, `feat(search): wire input`, `test(search): unit + e2e`).
2. Push branch; `gh pr create`.
3. **Write the PR description yourself** — today, not the agent. Reflect on: decisions, tradeoffs, where you steered.
4. Create `docs/agent-runs/day-2.md`: what went well, what drifted, where you pushed back. This becomes subagent training material tomorrow.

### Artifact for this step
- [Run log template](../06-artifacts.md#run-log-template) — copy it into `docs/agent-runs/day-2.md` and fill.

### Check
- CI green on the PR.
- Run log names two specific moments where supervision saved the run.

---

## Checkpoints
- You can name two specific moments where the agent almost went off-rails and what pulled it back.
- CI is green on the PR; PR is mergeable.

## Definition of Done
Feature merged to `main`. Tests pass. Run log committed.

## Cut-line
If time runs out: ship with unit tests only; skip the e2e for this feature (keep the e2e scaffold from Day 1).

## Mistakes to avoid
- Blindly accepting diffs. **Read every line.** If you didn't, the rest of the week is theater.
- Letting the agent expand scope. The agent always wants to "help"; you must pin it.
- Having the agent write both the code *and* the tests in the same session without scrutiny. Day 3 fixes this structurally.

## Signals
- **Over-controlling**: you rewrote the agent's code yourself more than twice. Fix: the brief is too vague OR `CLAUDE.md` is weak.
- **Under-controlling**: diff touched files outside the task scope, or tests pass but don't exercise the new behavior.

## Validation signal — you understand this if...
...from memory, you can describe the agent's *planning bias* (e.g., "wants to over-plan and tangent"), *edit bias* (e.g., "adds unrelated cleanups"), and *testing bias* (e.g., "tests match the implementation, not the spec"). These observations shape the subagent prompts on Day 3.

## References
- [Plan mode](https://docs.claude.com/en/docs/claude-code/plan-mode)
- [Anthropic: Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)

## Today's artifact bundle
- `docs/tasks/001-search.md` — the brief.
- `docs/agent-runs/day-2.md` — the run log (hand-written).
- Merged PR with feature, unit test, e2e.

Next: [Day 3 →](./day-3.md)
