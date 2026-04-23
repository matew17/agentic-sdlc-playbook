# Day 5 — CI-Triggered Agents via GitHub Actions

Nav: [Roadmap overview](./README.md) · [← Day 4](./day-4.md) · Next: [Day 6 →](./day-6.md)

## Objective
Move the reviewer and a new triage subagent **out of your laptop and into CI**. This is where "agent on my machine" becomes "agent in the SDLC."

## Concept primer (5 min)
Claude Code runs headlessly in GitHub Actions via `-p` (print mode). For PR review and CI triage, the agent should live in CI so it runs for everyone, always, in a clean environment — not contingent on your laptop being on.

Two workflows are immediately valuable:
1. **Review-on-PR** — runs the reviewer subagent on every PR and posts a comment.
2. **Triage-on-failure** — summarizes failing CI logs and suggests the fix.

A third (dep-hygiene) is a great demonstration of autonomous fire-and-forget work.

## Prerequisites
- Day 4 DoD met.
- `ANTHROPIC_API_KEY` added as a **repository secret** (Settings → Secrets → Actions).

## Time plan (3h)

| Step | Time | Outcome |
|---|---|---|
| 1 | 45 min | `review.yml` — agent reviews every PR |
| 2 | 45 min | `triage.yml` — agent summarizes failing CI |
| 3 | 30 min | `dep-hygiene.yml` — weekly autonomous dep PR |
| 4 | 30 min | Secret and permission hygiene |
| 5 | 30 min | End-to-end demo with a deliberate bug |

---

## Step 1 — Review-on-PR workflow (45 min)

### Do
Create `.github/workflows/review.yml`. The job checks out the PR, computes the diff against the base branch, runs Claude Code headlessly with the reviewer subagent, and posts the output as a PR comment via `gh pr comment`.

### Artifact for this step
- [`review.yml`](../06-artifacts.md#github-actions-starter) — copy the "agent-review" workflow verbatim.
- The [reviewer subagent](../06-artifacts.md#subagent-reviewer) from Day 3 must be present at `.claude/agents/reviewer.md` so the action can load it.

### Check
- Open a PR. Within ~1 minute, a comment appears with `Summary / Blocking / Suggestions / Nits`.
- The comment is attributed to GitHub Actions (not a user).
- No secrets leaked in the workflow log.

---

## Step 2 — Triage-on-failure workflow (45 min)

### Do
Create `.github/workflows/triage.yml`. Triggers on `workflow_run` of `ci` when `conclusion == 'failure'`. Pulls failing logs via `gh run view --log-failed`, summarizes root cause in ≤200 words, posts on the associated PR.

### Artifact for this step
- [`triage.yml`](../06-artifacts.md#github-actions-starter) — copy the "agent-triage" workflow. Adjust `head -n 400` if your logs are larger.

### Check
- Push a deliberately broken commit (e.g., introduce a type error). CI fails. Triage job runs and comments on the PR.
- Comment names the first error and proposes a minimal fix.

---

## Step 3 — Dep-hygiene workflow (30 min)

### Do
Create `.github/workflows/dep-hygiene.yml`. Scheduled weekly (`cron: '0 8 * * 1'`). Runs `pnpm outdated`, asks the agent to produce pnpm commands for **patch + minor only**, runs them, opens a PR if there are changes.

### Artifact for this step
- [`dep-hygiene.yml`](../06-artifacts.md#github-actions-starter) — copy verbatim.

### Check
- Run via `workflow_dispatch` manually. A chore PR either opens (if bumps exist) or the job exits cleanly.
- The PR body says "majors excluded." Verify nothing major snuck in.

---

## Step 4 — Secret and permission hygiene (30 min)

### Do
Audit each workflow's `permissions:` block. Least privilege:
- `review.yml`: `contents: read`, `pull-requests: write`.
- `triage.yml`: `contents: read`, `pull-requests: write`, `actions: read`.
- `dep-hygiene.yml`: `contents: write`, `pull-requests: write`.

Add `actions/dependency-review-action@v4` as a required check on PRs touching `package.json` (optional but recommended).

### Artifact for this step
- No new file; apply inline to each workflow. Refer back to [security subagent scope](../06-artifacts.md#subagent-security) — nothing in your workflow YAML should grant more than that subagent needs.

### Check
- `grep -r 'permissions:' .github/workflows` shows least-privilege blocks on every workflow.
- No `ANTHROPIC_API_KEY` literal anywhere except the repo secret reference.

---

## Step 5 — End-to-end demo (30 min)

### Do
Create a branch with a subtle bug (e.g., off-by-one in pagination). Open a PR. Verify the pipeline:
1. CI fails on e2e/test.
2. Triage subagent comments with the likely root cause.
3. You fix the bug; push.
4. CI goes green; reviewer subagent posts approval-style comment.
5. Merge via human approval.

### Artifact for this step
- `docs/agent-runs/day-5.md` using the [run log template](../06-artifacts.md#run-log-template). Paste the URLs of the review and triage comments.

### Check
- Three PR artifacts appeared automatically: CI status, reviewer comment, triage comment (on the red run).
- Human approval was the only "merge button" press.

---

## Checkpoints
- Every PR now attracts CI status + reviewer comment; red PRs also get triage.
- `CLAUDE.md` updated with a section: **"CI agent contract — what runs automatically on every PR."**
- Weekly dep-hygiene job verified manually once.

## Definition of Done
Three workflows shipped. End-to-end demo recorded in the run log. `CLAUDE.md` updated.

## Cut-line
Ship `review.yml` only. Triage and dep-hygiene move to Day 6 morning.

## Mistakes to avoid
- Running the agent with `Bash(*)` in CI. Scope permissions tightly in the headless prompt.
- Letting the reviewer agent approve its own PRs. Reviewer comments are informational; **humans merge**.
- Overformatting comment output. Ship plain structured markdown; iterate later.

## Signs you're overengineering
- Spent > 30 minutes on comment formatting.
- Built a "meta-orchestrator workflow" to coordinate the other workflows.
- More than 4 workflows this week.

## References
- [Claude Code GitHub Actions](https://docs.claude.com/en/docs/claude-code/github-actions)
- [Claude Code SDK / headless mode](https://docs.claude.com/en/docs/claude-code/sdk)

## Today's artifact bundle
- `.github/workflows/review.yml`
- `.github/workflows/triage.yml`
- `.github/workflows/dep-hygiene.yml`
- `CLAUDE.md` section: "CI agent contract"
- `docs/agent-runs/day-5.md` with demo run URLs

Next: [Day 6 →](./day-6.md)
