# Day 7 — Real Pilot Feature and Playbook Polish

Nav: [Roadmap overview](./README.md) · [← Day 6](./day-6.md)

## Objective
Ship **one real feature end-to-end through the full pipeline** with minimal intervention. Then polish and freeze the playbook. This is the demo you'll show your client.

## Concept primer (5 min)
It's not impressive that the agent wrote code. It's impressive that every piece is auditable, reviewed, tested, and gate-kept — and that you can narrate the whole thing in 15 minutes.

## Prerequisites
- Days 1–6 DoD met.
- You have one to two candidate features in mind.

## Time plan (3h)

| Step | Time | Outcome |
|---|---|---|
| 1 | 15 min | Pick the pilot feature; open GitHub issue |
| 2 | 30 min | `/start-feature` slash command |
| 3 | 105 min | Run the full pipeline (all 3 gates) |
| 4 | 30 min | Retro + playbook polish |

---

## Step 1 — Pick the pilot feature (15 min)

### Do
Pick a feature that touches UI + state + storage + tests. Recommended: **"Persistent favorites with localStorage, a dedicated `/favorites` route, and a keyboard shortcut (`F`) to toggle favoriting on the focused card."**

Open a GitHub issue describing it in one paragraph. Let the planner do the expansion.

### Artifact for this step
- Use the [task brief pattern](../05-playbook.md#task-brief-pattern-prompt-template) as the body of the issue so the planner has clean input.

### Check
- Issue open, labeled (`type:feature`).
- Autonomy level: `full` for everything except sensitive paths.

---

## Step 2 — `/start-feature` slash command (30 min)

### Do
Create `.claude/commands/start-feature.md`. Accepts an issue number; invokes planner → creates branch on approval → implementer → tester → reviewer → PR.

Also create `/review-pr` and `/triage-ci` while you're here.

### Artifact for this step
- [Slash commands](../06-artifacts.md#slash-commands) — copy all three (`start-feature.md`, `triage-ci.md`, `review-pr.md`).

### Check
- In Claude Code, typing `/` lists the three new commands.
- `/start-feature 1` (or whatever your issue number is) kicks off the planner.

---

## Step 3 — Run the full pipeline (105 min)

### Do
Execute the pipeline **end-to-end with the three human gates**:

**G1 — PRD approval.**
1. Planner produces `docs/prds/001-favorites.md`.
2. Read it. Push back at least once (insist on a missing acceptance criterion, e.g., "what happens if localStorage is disabled?").
3. When satisfied, flip frontmatter `approved: true`. Commit on a `docs/prd-favorites` branch, merge to `main`.

**Implementer + tester + reviewer loop.**
4. `/start-feature` continues. Orchestrator creates `feat/001-favorites` branch.
5. Implementer edits. Hooks run typecheck automatically.
6. Tester writes unit + e2e tests from the PRD. AC-to-test table in the run log.
7. Reviewer runs. Resolve any Blocking items (another implementer pass).
8. Orchestrator pushes branch. `gh pr create` with PR body from the run log.

**G2 — Pre-merge.**
9. CI runs: review.yml comments; ci.yml green.
10. You read the PR as you would any colleague's. Merge when satisfied.

**G3 — Production release.**
11. `deploy.yml` queues staging deploy automatically. Verify via local preview or actual staging URL.
12. Production deploy requires manual approval in GitHub Environments. Approve. Done.

### Artifact for this step
- Everything above references existing artifacts. The outputs of this step are:
  - `docs/prds/001-favorites.md` — the approved PRD.
  - `docs/agent-runs/001-favorites.md` — the run log.
  - Merged PR with feature, tests, and all subagent comments.

### Check
- Total human time on this feature ≤ 45 minutes (reading diff, writing PRD comments, approving gates). If you worked more, identify where autonomy failed.
- Every artifact is linkable from the PR: PRD, run log, reviewer comment, security comment (if applicable), CI, deploy record.

---

## Step 4 — Retro and playbook polish (30 min)

### Do
1. Write `docs/agent-runs/week-1-retro.md` against the five retro questions in [the roadmap index](./README.md#weekly-retro-sunday-20-minutes).
2. Skim [`05-playbook.md`](../05-playbook.md). Note anything you'd change based on the week. Edit the playbook (don't just list edits).
3. Trim `CLAUDE.md`. If anything there didn't earn its keep this week, delete it.
4. Remove unused subagents or skills. If `security` hasn't fired all week and your repo has no sensitive paths yet, park it (move to `.claude/agents/_parked/`).

### Artifact for this step
- `docs/agent-runs/week-1-retro.md`.
- Polished `05-playbook.md`, `CLAUDE.md`, `.claude/agents/`.

### Check
- A teammate could clone the repo, read `README.md` + `CLAUDE.md`, and ship the **next** feature solo. Mentally rehearse this walkthrough.

---

## Checkpoints
- One real feature shipped agentically, deployed through G3.
- Total human touch time ≤ 45 min.
- Retro committed. Playbook trimmed.

## Definition of Done
Feature merged **and** deployed to production. Week-1 retro committed. Playbook polished. All three slash commands work.

## Cut-line
Even a tiny feature counts. What matters is the **pipeline ran end-to-end with all three gates**. If deploy infrastructure isn't in place for your anime repo, simulate G3 with a staging-only deploy and document the simulation in the retro.

## Mistakes to avoid
- **Cramming a big feature.** Pick something you'd ordinarily ship in 90 minutes by hand; the agent should do it in 45 minutes of *your* time.
- **Skipping the retro.** The retro is the playbook update. No retro = no learning = week wasted.

## Signs you're ready
- You can hand the repo to a teammate and they can run the next feature solo with `README.md` + `CLAUDE.md`.
- You can narrate the whole pipeline to your client in 15 minutes using [`02-target-architecture.md`](../02-target-architecture.md) as the deck.

## References
- [`09-what-good-looks-like.md`](../09-what-good-looks-like.md) — the exam. Go through it tonight.

## Today's artifact bundle
- `docs/prds/001-favorites.md` (approved)
- `.claude/commands/start-feature.md`
- `.claude/commands/triage-ci.md`
- `.claude/commands/review-pr.md`
- Merged PR with full artifact chain
- `docs/agent-runs/001-favorites.md`
- `docs/agent-runs/week-1-retro.md`
- Polished `05-playbook.md` and `CLAUDE.md`

Done with Day 7. Go to the [exam](../09-what-good-looks-like.md).
