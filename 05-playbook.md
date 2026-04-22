# 05 — The Reusable Playbook

> This is what you carry between projects. Stack-agnostic. Tool-agnostic at the *shape* level, Claude-Code-opinionated at the *implementation* level.

---

## The seven-step pattern (applies to any SDLC agent workflow)

```
┌──────────────────────────────────────────────────────────────────────────┐
│ 1. SCOPE       Define the task boundary. What's in, what's out.          │
│ 2. GROUND      Ensure deterministic validation (tests, types, lint, CI). │
│ 3. AUTHORIZE   Decide what the agent may do. Who merges. Who deploys.    │
│ 4. DECOMPOSE   Plan → tasks → test strategy. Human approves the plan.    │
│ 5. EXECUTE     Agent loops: implement → test → self-review.              │
│ 6. VALIDATE    Humans + CI see an auditable artifact (PR + run log).     │
│ 7. CLOSE       Merge, deploy, observe. Retro feeds back into scope.      │
└──────────────────────────────────────────────────────────────────────────┘
```

Every step must have **an explicit artifact**. "Discussed in chat" doesn't count.

---

## Project onboarding checklist (run when joining a new codebase)

Use this when the client project starts. Allocate half a day.

- [ ] Verify there is `README.md` that explains how to run the project locally.
- [ ] Ensure `pnpm test`, `lint`, `typecheck`, `build` (or equivalent) exist and pass.
- [ ] Ensure CI exists and is a *required check* for merges.
- [ ] Identify the most brittle area of the codebase; agent will avoid it for now.
- [ ] Identify high-risk paths (auth, payments, migrations, configs, IaC). Mark for cautious mode.
- [ ] Draft `CLAUDE.md` v1 from the template. Commit on a branch; PR it; get human approval.
- [ ] Inventory existing humans in the PR review chain. Decide where agents fit (reviewer? implementer?).
- [ ] Inventory secrets. Confirm none are in repo. Add `gitleaks` if missing.
- [ ] Decide agent model tiers and set an initial weekly cost budget.
- [ ] Write ADR 0001: "Adopting agentic SDLC — scope, risks, success metrics."

Do not start authoring subagents before this list is complete.

---

## The autonomy decision matrix (pin this above your desk)

For any proposed automation, score each dimension 1–5. Sum tells you the posture.

| Dimension | 1 (low) | 3 | 5 (high) |
|---|---|---|---|
| **Reversibility** | Irreversible (prod deploy, migration, external email) | Revertable with work | Trivial revert (one commit) |
| **Blast radius** | Affects all users / all env | Team-scoped | Single file / local branch |
| **Ground truth strength** | No tests, vague spec | Partial tests | Strong tests + types |
| **Repeatability of task** | One-off, novel | Familiar pattern | Well-trodden |
| **Sensitivity of path** | Auth, payments, crypto | Business logic | Docs, UI tweaks |

- **≥ 20 points** → fully autonomous (no human in loop beyond PR merge).
- **13–19** → agent executes, human reviews PR and approves merge.
- **7–12** → cautious mode: agent drafts, human co-writes each change.
- **≤ 6** → human leads, agent assists (reviewer-only or planner-only).

---

## CLAUDE.md authoring rules (the highest-leverage artifact)

CLAUDE.md is the agent's **standing orders**. Treat it like a runbook, not a README.

**What belongs in CLAUDE.md**
1. Stack, package manager, required node version.
2. Canonical commands: install, dev, test, typecheck, lint, build.
3. Repo architecture in 5–10 lines. Where components, state, APIs, tests live.
4. **Rules**: MUST, MUST NOT, SHOULD, SHOULD NOT. Named and numbered.
5. The three human gates (link to playbook).
6. Preferred subagents for task types (planner → implementer → tester → reviewer).
7. Cautious-mode paths (authoritative list).
8. Pointers to `.claude/skills/` for reusable procedures.
9. Where to write run logs and ADRs.

**What does NOT belong**
- Company history, team org charts, Slack links.
- Tutorial content for humans.
- Things that change weekly (sprint goals, active tickets) — put these in the task brief, not CLAUDE.md.

**Sizing.** Aim for **200–400 lines**. Shorter → weak context. Longer → drift and token cost. Split into subdirectory `CLAUDE.md` files when a zone (e.g., `backend/`) has its own rules.

See the template in [`06-artifacts.md`](./06-artifacts.md#claudemd-template).

---

## The subagent design contract

Every subagent must answer these, in its system prompt, in order:

1. **Role.** One sentence.
2. **Inputs.** Exact files/paths/facts it expects.
3. **Outputs.** Exact files/paths it may write, exact format.
4. **Tools.** Explicit allowlist.
5. **Boundaries.** What it *must not* do.
6. **Success criteria.** How to know it's done.
7. **Escalation.** What to do when stuck.

If any of the 7 is vague, you will see drift. Common failures:
- Role too broad ("handles code") → narrow it.
- Outputs unspecified → agent invents structure and it varies per run.
- No escalation rule → agent either gives up silently or invents a wrong answer.

---

## Task brief pattern (prompt template)

Always use this shape when starting a task. Feed it to the orchestrator.

```
## Task
<one line>

## Context
- Repo: <name>
- Branch base: <main | develop>
- Related files: <list, or "planner: please find">
- Related docs: <prd link, adr link, design link>

## Acceptance
- Functional: <bullets>
- Non-functional: performance, accessibility, security
- Tests required: unit, e2e, which scenarios
- Out of scope: <be explicit>

## Authority
- Autonomy level: <full | cautious | reviewer-only>
- Paths off-limits: <list if any>
- Deadline: <if any>

## Gate plan
- G1 approver: <human>
- G2 approver: <human / team>
- G3: <N/A for feature | staging only | prod required>
```

This is the single most-reused artifact in your week. Copy it into every issue.

---

## The review ritual (every PR, without exception)

When the agent opens a PR, do this in order, always:

1. **Read the PR description.** Does the run log narrate what was done and why?
2. **Read the diff cold**, before reading the reviewer subagent's comments. Your first impressions are uncontaminated signal.
3. **Read the tests.** Are they testing *behavior against the spec*, or *the code's structure*? Only the former are useful.
4. **Check scope.** Did the diff touch anything outside the task?
5. **Check ignored concerns.** Accessibility? i18n? Error states? Loading states? Telemetry?
6. **Now read the reviewer subagent comments.** Do they align with yours? Divergence is signal — either the reviewer is weak or you missed something.
7. **Merge or push back.** Clear, specific feedback in comments, not vibes.

Rule of thumb: **If you find yourself "cleaning up" after the agent, stop and fix the root cause in CLAUDE.md, a subagent, or a skill.** Otherwise you're trapped babysitting.

---

## Intervention policy (write-it-down-once doc)

Put this in `docs/agentic-intervention-policy.md`:

**Always intervene** (don't let the agent do these unattended):
- Schema migrations or data backfills.
- Changes to auth, authz, crypto, signing, secrets.
- Payment flows or financial calculations.
- Deletes of production data or resources.
- Public API contract changes (breaking).
- Security patches labeled critical.
- IaC changes to networking, IAM, storage policies.

**Review before merge, agent may implement** (most things):
- Features, bug fixes, refactors, tests, deps, docs.

**Fire-and-forget okay** (truly autonomous):
- Dep patch bumps (if tests pass).
- Changelog generation.
- Stale issue labeling.
- Lint autofixes.
- Flaky-test triage comments.

This list is the single most politically important deliverable for your client. It is **both** permission and protection.

---

## When to iterate on a weak output (the repair loop)

The agent shipped a bad PR. Do not restart from zero. Repair:

1. **Tests weak?** Ask tester subagent to add cases: edge cases, failure modes, accessibility, loading/error states. *Name the missing cases explicitly.*
2. **Code drifted from spec?** Edit the PRD to be more precise; re-run implementer with "diff against new PRD."
3. **Scope bloat?** Comment on offending file: "revert this file, out of scope." Let agent apply the revert.
4. **Hallucinated dep/file?** Add a hook that runs `tsc` after every edit so the agent sees the error immediately.
5. **Bad pattern?** Codify it in a skill or in CLAUDE.md rules so it doesn't recur.
6. **Same failure repeats across runs?** The root cause is in your config, not the task. Stop and fix it.

**Rule:** every repair that is not a one-off must result in **a config or prompt change**, committed. If you're repeating corrections verbally, you are not learning — the agent is.

---

## Context hygiene

- **Scope per subagent.** Each subagent's CLAUDE.md or system prompt should include only what it needs.
- **Reset between tasks.** A fresh Claude Code session for each feature. Do not let context rot carry between features.
- **Pin files, don't dump files.** When you need the agent to consider file X, say "read `src/foo/bar.ts`" — do not paste its contents.
- **Use `/clear` generously** inside a long session. The context is a scratchpad, not a journal.

---

## Cost discipline

- Tier models by risk:
  - Haiku: planner drafts, reviewer, triage, dep bumps.
  - Sonnet: implementer, tester, most tasks.
  - Opus: only on second-attempt failures, architecture ADRs, complex refactors.
- Budget per task. Stop and ask for human help when exceeded.
- Weekly review: which subagent consumed what. Rebalance tiers.
- **No lift-and-shift from laptop to CI without re-checking cost.** A subagent run in Actions on every PR adds up fast.

---

## Graduation criteria (you are ready when...)

- You can onboard a new repo in under a half-day.
- Every feature ships via the same pipeline; no artisanal one-offs.
- You have zero "what did the agent do?" debates because run logs exist.
- You've had at least one save — a hook or reviewer caught a bad change before merge.
- A teammate can operate the system from `CLAUDE.md` and this playbook alone.

Next: [06-artifacts.md](./06-artifacts.md).
