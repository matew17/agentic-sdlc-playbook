# Day 6 — Production Hardening, Observability, Cost Discipline

Nav: [Roadmap overview](./README.md) · [← Day 5](./day-5.md) · Next: [Day 7 →](./day-7.md)

## Objective
Make the system **inspectable, cheap, and resilient**. This is the difference between a demo and something you'd defend in a post-mortem.

## Concept primer (5 min)
Observability in agent systems is a **content problem, not a metrics problem**. Run logs + ADRs beat dashboards because they encode *why*, not just *what*. Add metrics once you have a pattern; never the other way around.

## Prerequisites
- Day 5 DoD met. CI agents work.

## Time plan (3h)

| Step | Time | Outcome |
|---|---|---|
| 1 | 45 min | Run log convention + backfill |
| 2 | 30 min | ADR 0001 — the architecture decision |
| 3 | 45 min | Secret scanning + security subagent |
| 4 | 30 min | Cost discipline — model tiers and budgets |
| 5 | 30 min | Failure drill — three induced failures |

---

## Step 1 — Run log convention and backfill (45 min)

### Do
Create `docs/agent-runs/TEMPLATE.md`. Update `CLAUDE.md`: rule "every task ends with a run log in `docs/agent-runs/YYYY-MM-DD-<slug>.md`, linked from the PR body." Backfill run logs for Days 1–5 if they don't exist yet.

### Artifact for this step
- [Run log template](../06-artifacts.md#run-log-template). Copy as `docs/agent-runs/TEMPLATE.md`.
- Update the rule in your [`CLAUDE.md`](../06-artifacts.md#claudemd-template) (R3 in the template).

### Check
- `docs/agent-runs/` has at least one real entry per merged PR of the week.
- Each PR description links its run log.

---

## Step 2 — ADR 0001 (30 min)

### Do
Write `docs/adr/0001-agentic-sdlc-architecture.md` documenting the architecture from [`02-target-architecture.md`](../02-target-architecture.md). Include: context, decision, consequences, rollback. Status: `accepted`.

Add a rule to `CLAUDE.md`: "any decision that changes architecture triggers an ADR via the planner." From today, the planner creates `docs/adr/NNNN-*.md` when it detects an architectural proposal.

### Artifact for this step
- Use the [ADR prompt](../06-artifacts.md#reusable-prompts) ("Write an ADR") to scaffold the file. Then edit for content fidelity.

### Check
- ADR exists in `docs/adr/0001-*.md` with `status: accepted`.
- `CLAUDE.md` references `docs/adr/` as the home for architectural decisions.

---

## Step 3 — Secret scanning + security subagent (45 min)

### Do
1. Add `gitleaks` as both a pre-commit hook and a GitHub Action. Block commits/PRs with leaked secrets.
2. Add `pnpm audit` (or `osv-scanner`) to CI.
3. Create `.claude/agents/security.md`. Runs on PRs touching auth/payments/crypto patterns. Produces a structured comment with CRITICAL / HIGH / MEDIUM / LOW / INFO findings.
4. Extend `review.yml` to additionally run the security subagent when the diff matches `src/(auth|payments|crypto)/**` — or wire a new `security.yml` workflow.

### Artifact for this step
- [Subagent — security](../06-artifacts.md#subagent-security). Copy to `.claude/agents/security.md`.
- Gitleaks install: `brew install gitleaks`; pre-commit config: `gitleaks protect --staged`.
- `pnpm audit` line in `ci.yml` (add after `pnpm install`).

### Check
- Commit a fake AWS key to a throwaway branch; gitleaks blocks it. Remove it. Commit passes.
- On a PR that touches an imagined `src/auth/` file, the security subagent comment appears.

---

## Step 4 — Cost discipline (30 min)

### Do
Open each subagent file and set explicit `model:` tiers. Default policy:
- Haiku: `planner` (drafts), `reviewer`, `triage`.
- Sonnet: `implementer`, `tester`, `security`.
- Opus: **only** for retry after two failures or for heavy architecture work.

Add a `## Context budget` note to each subagent file with a soft token ceiling. The orchestrator prompt in `CLAUDE.md` should say: "if a subagent exceeds budget, stop and ask the human."

Set a calendar reminder for a **weekly 10-minute cost review** — read the Anthropic usage dashboard, note any subagent that drifted high.

### Artifact for this step
- [Cost discipline section in the playbook](../05-playbook.md#cost-discipline). Cite it in `CLAUDE.md`.
- Edit the frontmatter of each subagent file in `.claude/agents/` to set `model:` correctly.

### Check
- Every subagent file has an explicit `model:` line.
- `CLAUDE.md` references the cost policy.

---

## Step 5 — Failure drill (30 min)

### Do
Induce three failures. Observe recovery. Log everything.

1. **Merge conflict.** Create a branch that conflicts with a concurrent change on `main`. Ask the orchestrator to resolve. Watch how it handles it — does it stop and ask? Retry cleanly? Escalate?
2. **Failing e2e.** Break a selector. Let the full pipeline run. Triage comments. Reviewer flags.
3. **Hallucinated file path.** Prompt: "update `src/utils/notARealFile.ts`." The implementer should either search and ask, or stop. If it silently invents a file, your implementer prompt's "before editing, confirm symbols exist" rule is too soft — tighten it.

### Artifact for this step
- `docs/agent-runs/day-6-failure-drill.md` from the [run log template](../06-artifacts.md#run-log-template) — include a `## Failures and retries` section with each drill.

### Check
- None of the three failures corrupted repo state, leaked secrets, or pushed bad code to `main`.
- You made at least one config fix based on what you observed (prompt tweak, hook, rule).

---

## Checkpoints
- Run logs exist for every PR merged this week.
- First ADR committed.
- `pnpm audit` and gitleaks run in CI.
- Every subagent file has a `model:` tier and a context budget.
- Failure drill documented.

## Definition of Done
Run log template + backfilled entries; ADR 0001; security subagent + gitleaks + audit; cost tiers on every subagent; failure drill logged with at least one config fix committed.

## Cut-line
Skip cost tiering and failure drill step 3. Run logs, ADR, gitleaks, and at least one failure drill must ship.

## Mistakes to avoid
- **Metrics theater.** A dashboard you don't check is waste. Commit to a *weekly written review*.
- **Optional ADRs.** If decisions aren't recorded, you cannot debug them six weeks later.
- **Security subagent with edit tools.** Same rule as the reviewer — no silent fixes.

## Signs you're under-controlling
- After a failing run, you cannot reconstruct what the agent tried without scrolling chat history.
- No one on the team can name the cost of a typical task run to an order of magnitude.

## References
- [MCP servers](https://docs.claude.com/en/docs/claude-code/mcp) — skim only; useful for Day 6+ advanced integrations.
- [gitleaks](https://github.com/gitleaks/gitleaks)
- [Anthropic usage dashboard](https://console.anthropic.com/settings/usage)

## Today's artifact bundle
- `docs/agent-runs/TEMPLATE.md`
- `docs/adr/0001-agentic-sdlc-architecture.md`
- `.claude/agents/security.md`
- `gitleaks` pre-commit hook + CI step
- `pnpm audit` in `ci.yml`
- Updated subagent frontmatter with `model:` tiers
- `CLAUDE.md` additions: R3 run-log rule, cost policy reference, ADR policy
- `docs/agent-runs/day-6-failure-drill.md`

Next: [Day 7 →](./day-7.md)
