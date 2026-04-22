# 07 — Failure Modes and Mitigations

> These are the failures I've seen chew up real teams. Memorize the pattern and the rail that stops it.

---

## Taxonomy

Agentic SDLC failures fall into five families. Each one has a canonical mitigation.

1. **Drift** — agent slowly diverges from the task.
2. **Hallucination** — agent invents facts, files, APIs.
3. **Overreach** — agent does more than asked.
4. **Silent wrongness** — output looks right; isn't.
5. **Over-reliance** — humans stop thinking; agent becomes the bottleneck of trust.

---

## 1. Drift

### 1a. Scope creep
**Symptom.** Diff touches files unrelated to the PRD.
**Root cause.** Vague task brief; agent pattern-matches to "things that look like they need fixing."
**Mitigation.**
- PRD with explicit `## Out of scope`.
- Reviewer subagent checks scope as a Blocking dimension.
- Hook: pre-commit rejects if diff touches more than N files without a `scope-expand` flag in the run log.

### 1b. Prompt rot in long sessions
**Symptom.** Agent "forgets" earlier rules; edits degrade over time.
**Root cause.** Context window pressure; earlier messages compressed away.
**Mitigation.**
- Fresh session per feature.
- `/clear` aggressively.
- Critical rules live in `CLAUDE.md`, not the chat.

### 1c. Architectural drift across features
**Symptom.** Week 3: no two features use the same pattern.
**Root cause.** No ADRs; no canonical patterns codified in skills.
**Mitigation.**
- ADR per architectural decision, linked from `CLAUDE.md`.
- Skill files for canonical patterns.
- Reviewer subagent cites CLAUDE.md rules in comments.

---

## 2. Hallucination

### 2a. Imaginary files / functions / libraries
**Symptom.** Code references `utils/helpers.ts` which doesn't exist, or a function that doesn't.
**Root cause.** Agent inferred what *should* exist rather than reading what does.
**Mitigation.**
- Hook: post-edit `tsc` auto-runs; agent sees failures immediately.
- Instruction in implementer subagent: "before editing, `Grep` to confirm referenced symbols exist."
- CI typecheck is a required check.

### 2b. Imaginary APIs on real libraries
**Symptom.** Calls `axios.createSafe(...)` — plausible, not real.
**Root cause.** Model confabulates plausible APIs.
**Mitigation.**
- Typecheck catches most.
- Skill file for each major library lists the actually-used surface.
- For new deps, implementer must read the changelog / readme in the run log before use.

### 2c. Imaginary test passes
**Symptom.** Agent claims tests pass; they were never run.
**Root cause.** Agent summarized without executing; or test runner was misconfigured.
**Mitigation.**
- Required CI check runs tests in a clean env.
- Run log must include a command line and a truncated stdout excerpt.

---

## 3. Overreach

### 3a. Unauthorized force push / main edits
**Symptom.** History rewritten. Panic ensues.
**Root cause.** Broad shell allowlist.
**Mitigation.**
- Deny-list `git push --force*`, `git push origin main`, `git merge *`, `gh pr merge *` in `.claude/settings.json`.
- Branch protection on `main` (required reviews, required checks, no force-push).

### 3b. Secret exposure
**Symptom.** `.env` committed; API key in test fixture.
**Root cause.** No path guard; no scanner.
**Mitigation.**
- Hook blocks edits in `.env*` without explicit flag.
- `gitleaks` pre-commit + CI.
- Dependabot/Secret scanning enabled on the repo.

### 3c. Unsafe external calls
**Symptom.** Agent runs `curl | bash`, installs a random package.
**Root cause.** Broad network/install permission.
**Mitigation.**
- Scoped Bash allowlist — no raw `curl`, `wget`, `bash -c` without explicit approval.
- New deps require a run-log entry; reviewer subagent flags new deps.

---

## 4. Silent wrongness

### 4a. Tests match the bug
**Symptom.** Suite green; production broken.
**Root cause.** Same reasoning wrote code and tests.
**Mitigation.**
- **Tester subagent writes tests from PRD only**, ideally before seeing the code.
- Mutation testing on high-risk paths (e.g., `stryker` for JS) during review.
- E2E for every user-facing flow.

### 4b. Plausible but wrong output
**Symptom.** Code looks reasonable; behavior off by one / wrong ordering / wrong units.
**Root cause.** Model pattern-matched rather than reasoned.
**Mitigation.**
- Property-based tests on pure functions.
- E2E for behavior.
- PR description *must explain behavior* in prose; divergence between prose and code is often the bug.

### 4c. Green-is-meaningless CI
**Symptom.** CI passes but coverage is thin or irrelevant.
**Root cause.** No enforced coverage gate; tests only cover happy paths.
**Mitigation.**
- Minimum coverage thresholds on new files.
- Required: at least one negative-path test per feature.
- Periodic (monthly) manual sample-review of auto-generated tests.

---

## 5. Over-reliance and cultural drift

### 5a. Rubber-stamp PRs
**Symptom.** Humans click "approve" without reading.
**Root cause.** Review fatigue; no visible signal of what changed conceptually.
**Mitigation.**
- PRs < 400 lines of diff. Agent splits otherwise.
- Run log at the top of PR body — always.
- Reviewer subagent comment *disagrees* with itself periodically (ask it to find two things to improve even on clean diffs). Keeps reviewers alert.
- Spot-audit: one PR per week gets a deep human-only review; findings feed back.

### 5b. Skills atrophy
**Symptom.** Juniors can't debug without the agent; seniors can't remember the code.
**Root cause.** Too much autonomy too quickly.
**Mitigation.**
- "Agent-off Fridays" — one day a week, certain tasks done by hand.
- Require engineers to write the run log prose, not agent-generated, for features they personally drive.

### 5c. Compliance / audit gaps
**Symptom.** Auditor asks "who approved this?"; logs are ephemeral chat.
**Root cause.** No durable audit trail for agent decisions.
**Mitigation.**
- Run logs in git (can never be deleted silently).
- ADRs for anything architectural.
- PR description is the audit record. It must be complete.

---

## The debugging rubric

When something goes wrong, ask in order:

1. **Which gate failed?** G1 (spec), G2 (PR), G3 (release)? Or none — a silent failure?
2. **Which subagent produced the faulty output?**
3. **Was the faulty output caught by ground truth (tests, tsc, lint) or only by human review?**
4. **What change to CLAUDE.md, subagent, skill, or hook would have caught this automatically?**
5. **Make that change. Commit it. Link it in the run log as "fix-forward from incident N."**

If step 5 is skipped, you're repairing by hand. You're teaching yourself, not the system.

---

## Kill switches

You must have at least these three:

- **Repo-level:** `gh api --method PATCH /repos/:owner/:repo -f allow_auto_merge=false` + disable Actions in settings. Two minutes to enact.
- **Workflow-level:** `workflow_dispatch` on `manual-stop.yml` that updates a repo variable `AGENT_DISABLED=1`; all agent workflows check it and exit early.
- **Local-level:** A shell alias that removes `.claude/agents/*` and reverts to bare Claude Code if you need to stop mid-run.

Write these down. Practice firing them once.

---

## Known unknowns

Things you should expect to hit that this playbook doesn't fully solve:

- **Long-running tasks (>20 min)** will run out of patience/context. Chunk them.
- **Multi-repo changes** are poorly served by a single-repo-scoped agent. Use a monorepo or a coordinating human.
- **Stateful dev environments** (DB migrations, seeded data) need careful sandboxing. Never let the agent touch shared dev DBs.
- **Non-trivial UI polish** (pixel-perfect, animations) is still better done by humans. Don't force agentic flows where they don't earn their keep.

Next: [08-tool-transfer.md](./08-tool-transfer.md).
