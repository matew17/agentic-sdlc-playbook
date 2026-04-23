# Day 4 — Hooks, Permissions, Human Gates, and Cautious Mode

Nav: [Roadmap overview](./README.md) · [← Day 3](./day-3.md) · Next: [Day 5 →](./day-5.md)

## Objective
Formalize the authority model. Make it **impossible** for the agent to violate your rules by accident.

## Concept primer (5 min)
- **Permissions** (`.claude/settings.json`) define the allowlist/denylist of tools and paths. This is your static enforcement.
- **Hooks** are shell commands Claude Code runs around tool calls (`PreToolUse`, `PostToolUse`, `UserPromptSubmit`). This is your dynamic enforcement.
- **Human gates** are workflow boundaries — not prompts. They live in branch protection, env frontmatter, and GitHub Environments.

A system with roles but no enforcement is aspirational. A system with enforcement is operational.

## Prerequisites
- Day 3 DoD met.
- `jq` installed (hooks use it to parse JSON): `brew install jq` or equivalent.

## Time plan (3h)

| Step | Time | Outcome |
|---|---|---|
| 1 | 45 min | `.claude/settings.json` permissions baseline |
| 2 | 45 min | Three hooks installed and executable |
| 3 | 45 min | Three human gates codified (G1, G2, G3) |
| 4 | 45 min | Adversarial test: deliberately try to break the rules |

---

## Step 1 — Permissions baseline (45 min)

### Do
Create `.claude/settings.json` with strict allow/deny. Scope every `Bash(...)` to the narrowest pattern. Deny force-push, main push, merge, rm -rf.

### Artifact for this step
- [Hooks and settings](../06-artifacts.md#hooks-and-settings) — the full `settings.json`. Copy verbatim; adjust to your scripts.

### Check
- Inside Claude Code, try `Bash(rm -rf node_modules)` manually — it should be blocked.
- Try `Bash(git push origin main)` — blocked.

---

## Step 2 — Install hooks (45 min)

### Do
Create three hooks, all executable (`chmod +x`):

1. **`block-sensitive-paths.sh`** — `PreToolUse` for `Edit|Write`. Blocks edits in `auth/`, `payments/`, `db/migrations/`, `.github/workflows/`, `.env*` unless `CC_ALLOW_SENSITIVE=1`. This is **cautious mode**.
2. **`typecheck-on-edit.sh`** — `PostToolUse` for `Edit|Write`. Runs `pnpm typecheck` when a `.ts`/`.tsx`/`.vue` is touched; injects the result into context.
3. **`inject-context.sh`** — `UserPromptSubmit`. Injects current date + current branch into the prompt. Reduces drift.

### Artifact for this step
- [Hooks and settings](../06-artifacts.md#hooks-and-settings) — all three scripts are there. Place in `.claude/hooks/`.
- Then wire them into `.claude/settings.json` under `hooks`.

### Check
- Try to edit `src/auth/login.vue` (even if the dir doesn't exist yet — create a fake one): must be blocked without `CC_ALLOW_SENSITIVE=1`.
- Edit any `.ts` file: watch `tsc` output get added to agent context.

---

## Step 3 — Codify the three human gates (45 min)

### Do

**G1 — PRD sign-off.** Rule: the implementer will refuse to run unless `docs/prds/<id>.md` frontmatter has `approved: true`. Enforce this via a CLAUDE.md rule + a pre-implementer hook that greps the frontmatter and fails if `approved: false`.

**G2 — Pre-merge review.** Already partly in place (branch protection + required reviewer). Today: add `docs/agentic-intervention-policy.md` (from the playbook) so the gate is documented.

**G3 — Production release.** Create GitHub Environment named `production` in repo settings. Require reviewer. Update `.github/workflows/deploy.yml` (Day 5 wires the actual deploy).

### Artifact for this step
- [Intervention policy](../05-playbook.md#intervention-policy-write-it-down-once-doc) — copy into `docs/agentic-intervention-policy.md`.
- [Authority model](../02-target-architecture.md#the-authority-model-memorize-this) — cite from `CLAUDE.md`.
- [`deploy.yml`](../06-artifacts.md#github-actions-starter) — the production job gated by the `production` environment.

### Check
- Try to invoke implementer on a PRD with `approved: false` — it refuses.
- GitHub Environments shows `production` with required reviewer.

---

## Step 4 — Adversarial test (45 min)

### Do
Deliberately try to break each rule. Every attempt should fail cleanly. Log all attempts and results in `docs/agent-runs/day-4.md`.

1. Ask the agent: "edit `.github/workflows/ci.yml` to remove the typecheck step." → blocked by hook.
2. Ask: "push this directly to main." → blocked by deny list + branch protection.
3. Ask: "start implementing the favorites PRD" without approval. → implementer refuses.
4. Ask: "remove `node_modules` so we can reinstall." → blocked.
5. Set `CC_ALLOW_SENSITIVE=1`, edit a sensitive path, confirm the hook *allows* it with the flag and logs an override.

### Artifact for this step
- `docs/agent-runs/day-4.md` from the [run log template](../06-artifacts.md#run-log-template). Add a `## Adversarial attempts` section listing what you tried and the outcome.

### Check
- All five attempts behave as expected. The override path (step 5) is clearly recorded.

---

## Checkpoints
- `git log` on `main` has no agent-authored commits that didn't come through a PR.
- You can articulate the authority model (who does what) without looking.
- Attempted violations are blocked automatically, not by hope.

## Definition of Done
`.claude/settings.json`, three hooks (executable and wired), `docs/agentic-intervention-policy.md`, GitHub `production` env, adversarial-test run log — all committed on `main`.

## Cut-line
Skip the `inject-context.sh` hook. Keep the denylist and the `block-sensitive-paths.sh` + `typecheck-on-edit.sh` hooks.

## Mistakes to avoid
- **Allowlisting `Bash(*)`**. That's root access on your machine. Always scope.
- **Only using hooks for logging.** Hooks are *enforcement*; logging is a side effect.
- Writing long intervention policies. Keep the list short and enforce it.

## Signs you're under-controlling
- You can't answer: "If the agent goes rogue tonight, what's the worst it can do?"
- Your denylist has fewer than 5 entries.
- You've never tested `CC_ALLOW_SENSITIVE=0 -> blocked, =1 -> allowed`.

## References
- [Hooks](https://docs.claude.com/en/docs/claude-code/hooks)
- [Settings / permissions](https://docs.claude.com/en/docs/claude-code/settings)

## Today's artifact bundle
- `.claude/settings.json`
- `.claude/hooks/block-sensitive-paths.sh`
- `.claude/hooks/typecheck-on-edit.sh`
- `.claude/hooks/inject-context.sh`
- `docs/agentic-intervention-policy.md`
- GitHub `production` env config (external to repo; note it in the run log)
- `docs/agent-runs/day-4.md` with adversarial-test results

Next: [Day 5 →](./day-5.md)
