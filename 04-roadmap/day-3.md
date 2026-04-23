# Day 3 — Subagents and Skills: The Role Carve-Up

Nav: [Roadmap overview](./README.md) · [← Day 2](./day-2.md) · Next: [Day 4 →](./day-4.md)

## Objective
Replace the single-agent loop with specialist subagents and one reusable skill. Every subagent has one job, narrow tools, and a clear contract.

## Concept primer (5 min)
- **Subagent** = scoped, system-prompted agent invoked by the main session. Gets its own context, its own tool allowlist, optionally its own model.
- **Skill** = reusable procedure (markdown + scripts) loaded on demand.
- Subagents are **who**; skills are **how**.

The single biggest quality lift in agentic SDLC work comes from **separating the writer from the critic**. That's why tester and reviewer are separate subagents from implementer.

## Prerequisites
- Day 2 DoD met. You have run logs and intuition about where the single-agent flow wobbled.

## Time plan (3h)

| Step | Time | Outcome |
|---|---|---|
| 1 | 20 min | Planner subagent |
| 2 | 20 min | Implementer subagent |
| 3 | 20 min | Tester subagent |
| 4 | 15 min | Reviewer subagent |
| 5 | 20 min | Orchestrator wiring in `CLAUDE.md` |
| 6 | 45 min | `vue-component` skill |
| 7 | 20 min | Re-run a small task through the pipeline |

---

## Step 1 — Planner subagent (20 min)

### Do
Create `.claude/agents/planner.md`. Narrow tools (`Read`, `Write`, `Edit`, `Glob`, `Grep`). Output restricted to `docs/prds/`.

### Artifact for this step
- [Subagent — planner](../06-artifacts.md#subagent-planner). Copy the full frontmatter + body. Adjust `model` (Haiku is fine).
- Also create `docs/prds/TEMPLATE.md` from the [PRD template](../06-artifacts.md#prd--implementation-brief-template) so the planner has a canonical shape to emit.

### Check
- Run `/agents` in Claude Code; the planner is listed.
- Call it: "planner: produce a PRD for a favorites list (localStorage, top bar icon, dedicated route)."
- Output lands in `docs/prds/<slug>.md` with `approved: false` frontmatter.

---

## Step 2 — Implementer subagent (20 min)

### Do
Create `.claude/agents/implementer.md`. Tools include `Bash` but **only** pnpm/git scoped by permissions (you enforce on Day 4).

### Artifact for this step
- [Subagent — implementer](../06-artifacts.md#subagent-implementer). Copy verbatim. Model: Sonnet.
- Create `docs/templates/task-decomposition.md` from the [task decomposition template](../06-artifacts.md#task-decomposition-template) so the implementer knows what a decomposed plan looks like.

### Check
- Implementer refuses to write tests or edit docs when asked. If it doesn't, tighten the **must-nots** section.

---

## Step 3 — Tester subagent (20 min)

### Do
Create `.claude/agents/tester.md`. Tools restricted to writing `**/*.spec.ts` and reading source. **Critical rule**: tester writes from the PRD, not from the implementation diff.

### Artifact for this step
- [Subagent — tester](../06-artifacts.md#subagent-tester).
- Reference the [test strategy template](../06-artifacts.md#test-strategy-template) in the tester prompt — insist on AC → test mapping.

### Check
- Ask the tester to write tests for an imagined feature whose PRD you just approved. It should produce the AC-to-test table in the run log even when the implementation isn't done yet.

---

## Step 4 — Reviewer subagent (15 min)

### Do
Create `.claude/agents/reviewer.md`. **No edit tools.** Only read + bash for running checks.

### Artifact for this step
- [Subagent — reviewer](../06-artifacts.md#subagent-reviewer).
- Create `docs/templates/pr-review-checklist.md` from the [PR review checklist](../06-artifacts.md#pr-review-checklist) — the reviewer will cite it.

### Check
- Ask reviewer to review a recent commit. Output lands as structured markdown with Summary / Blocking / Suggestions / Nits sections. No edits.

---

## Step 5 — Orchestrator wiring (20 min)

### Do
Update `CLAUDE.md`:
- Add a **Subagent routing** section: which task types go to which subagent.
- Add a rule: "for feature tasks, call planner → wait for `approved: true` → implementer → tester → reviewer."
- Add: "do not skip steps."

### Artifact for this step
- The **Subagent routing** block in the [CLAUDE.md template](../06-artifacts.md#claudemd-template). Merge into your existing `CLAUDE.md`.

### Check
- New `CLAUDE.md` is committed. Subagent files visible in `.claude/agents/`.

---

## Step 6 — `vue-component` skill (45 min)

### Do
Create `.claude/skills/vue-component/SKILL.md`. Capture your opinions: SFC structure, props typing, composables split, testing approach, anti-patterns.

### Artifact for this step
- [Skill example — `vue-component`](../06-artifacts.md#skill-example--vue-component). Copy verbatim; adjust to your anime-repo conventions.

### Check
- Ask implementer to create a new `.vue` file; it references the skill's structure rules in its output.

---

## Step 7 — Re-run a small task through the pipeline (20 min)

### Do
Pick a tiny task ("add an empty state when the catalog is empty"). Run it end-to-end: planner → implementer → tester → reviewer. Time how long each phase took; note cost if visible.

### Artifact for this step
- `docs/agent-runs/day-3.md` — [Run log template](../06-artifacts.md#run-log-template). Record the AC→test mapping, any reviewer Blocking items, and how long each subagent took.

### Check
- The reviewer found **at least one real issue** the single-agent flow on Day 2 missed. If it didn't, your reviewer prompt is too soft — tighten it (add bias toward skepticism).

---

## Checkpoints
- Four subagents in `.claude/agents/`; invokable via `/agents`.
- One skill in `.claude/skills/vue-component/`.
- Run log for today shows the pipeline ran and the reviewer caught something.

## Definition of Done
Planner, implementer, tester, reviewer subagents committed. `vue-component` skill committed. One task run through the new pipeline. Run log committed.

## Cut-line
Ship planner + reviewer only today; implementer and tester tomorrow morning. **Never skip the reviewer** — that's the quality lift.

## Mistakes to avoid
- **Overlapping responsibilities.** Each subagent's job must fit in one sentence. If two are vague, merge or narrow.
- **Giving the reviewer `Edit` access.** It will quietly fix things and you'll never learn what was wrong.
- **Skills that are just prose.** Skills shine when they include commands/templates the agent actually runs.

## Signs you're overengineering
- More than 5 subagents on Day 3.
- Negotiation protocols between subagents.
- Anything resembling "agent-to-agent chat."

## References
- [Subagents docs](https://docs.claude.com/en/docs/claude-code/sub-agents)
- [Skills docs](https://docs.claude.com/en/docs/agents-and-tools/skills)

## Today's artifact bundle
- `.claude/agents/planner.md`
- `.claude/agents/implementer.md`
- `.claude/agents/tester.md`
- `.claude/agents/reviewer.md`
- `.claude/skills/vue-component/SKILL.md`
- `docs/prds/TEMPLATE.md`
- `docs/templates/task-decomposition.md`
- `docs/templates/pr-review-checklist.md`
- `CLAUDE.md` with updated Subagent routing
- `docs/agent-runs/day-3.md`

Next: [Day 4 →](./day-4.md)
