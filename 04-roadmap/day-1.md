# Day 1 — Foundations and a Ruthless Repo Baseline

Nav: [Roadmap overview](./README.md) · Next: [Day 2 →](./day-2.md)

## Objective
Before any agent runs, the repo must have **ground truth**. Agents without tests/lint/typecheck are a loaded footgun.

## Concept primer (5 min)
Every agentic workflow has three layers:
1. **Capability** — what the agent can do.
2. **Ground truth** — what *deterministically* validates its output.
3. **Authority** — what it's allowed to change.

Day 1 builds layer 2. No agent work until ground truth exists.

## Prerequisites
- Anime-catalog Vue 3 + Vite + TS repo pushed to GitHub.
- pnpm installed; Node 20+.
- `gh` CLI authenticated.
- Claude Code installed locally (`npm i -g @anthropic-ai/claude-code` or per platform).

## Time plan (3h)

| Step | Time | Outcome |
|---|---|---|
| 1 | 30 min | Claude Code installed + `/init` run |
| 2 | 60 min | Ground-truth tooling (vitest, playwright, eslint, typecheck, scripts) |
| 3 | 45 min | `ci.yml` workflow passing on PR |
| 4 | 30 min | Branch protection enabled |
| 5 | 15 min | `CLAUDE.md` v1 committed |

---

## Step 1 — Install and verify Claude Code (30 min)

### Do
1. Install Claude Code; log in.
2. `cd` into the anime repo; run `claude`.
3. Use `/init` to generate an initial `CLAUDE.md` scaffold. Commit it on a branch `chore/claude-bootstrap`.
4. Poke around plan mode (`Shift+Tab` → `plan`) to get a feel — no edits yet.

### Artifact for this step
- Generated `CLAUDE.md` stub (you'll replace it in Step 5 using the template).

### Check
- `claude` opens and recognizes your project.
- `git status` shows a pending `CLAUDE.md` ready to commit on a branch.

---

## Step 2 — Ground-truth tooling (60 min)

### Do
Install and wire four deterministic checks.

```bash
pnpm add -D vitest @vitest/ui @vue/test-utils jsdom
pnpm add -D @playwright/test && pnpm exec playwright install --with-deps chromium
pnpm add -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin eslint-plugin-vue vue-eslint-parser
```

Add scripts to `package.json`:

```json
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "typecheck": "vue-tsc --noEmit",
  "lint": "eslint . --ext .ts,.vue",
  "test": "vitest run",
  "test:watch": "vitest",
  "test:e2e": "playwright test"
}
```

Create one trivial unit test (e.g. `src/utils/__tests__/slug.spec.ts`) and one e2e scaffold (`e2e/home-loads.spec.ts` that navigates to `/` and asserts the app mounts).

Create minimal `eslint.config.js` (flat) or `.eslintrc.cjs` with `@typescript-eslint` and `plugin:vue/vue3-recommended`.

### Artifact for this step
- Not in `06-artifacts.md` by intention — this is boilerplate wiring. **Do not over-engineer this step.** Keep the one unit test trivial, the one e2e test navigational.

### Check
```bash
pnpm typecheck && pnpm lint && pnpm test && pnpm test:e2e && pnpm build
```
All green. Commit: `chore: ground-truth tooling (vitest, playwright, eslint, tsc)`.

---

## Step 3 — GitHub Actions CI (45 min)

### Do
Create `.github/workflows/ci.yml`. Keep it tight; skip e2e here (its own workflow later on day 5).

### Artifact for this step
- [`ci.yml` from the GitHub Actions starter](../06-artifacts.md#github-actions-starter) — copy the `ci` job. You can leave out the `e2e` job today and wire it Day 5.

### Check
- Open a trivial PR (edit README). CI runs, passes, PR gets a green check.
- Take a screenshot or copy the run URL into `docs/agent-runs/day-1.md` (you'll create this doc at end of day).

---

## Step 4 — Branch protection (30 min)

### Do
In GitHub → **Settings → Branches → Add rule** for `main`:
- Require pull request before merging.
- Require status checks to pass: `ci`.
- Require branches to be up to date.
- Disable force pushes. Disable deletions.

Add yourself as a required reviewer for now — the **reviewer subagent** will take over this slot on day 5.

### Artifact for this step
- Reference: [authority model table in `02-target-architecture.md`](../02-target-architecture.md#the-authority-model-memorize-this). Day 1 enforces one row: *no one pushes to `main` directly*.

### Check
- Try `git push origin main` from a throwaway branch. It must be rejected.

---

## Step 5 — Author `CLAUDE.md` v1 (15 min)

### Do
Replace the `/init` stub with the full template, adapted for your repo.

### Artifact for this step
- [`CLAUDE.md` template](../06-artifacts.md#claudemd-template). Copy it, fill stack/commands/architecture for your anime repo, commit.

### Check
- `CLAUDE.md` is 200–400 lines, has MUST and MUST-NOT sections, routes to subagents by task type (even though subagents don't exist yet — you'll create them on Day 3).
- Commit to `chore/claude-bootstrap`, open PR, CI green, merge.

---

## Checkpoints
- `pnpm typecheck && pnpm lint && pnpm test && pnpm build` all pass locally.
- A PR triggers CI; CI is required to merge.
- `CLAUDE.md` is committed to `main`.

## Definition of Done
Clean repo + passing CI + branch protection + `CLAUDE.md` v1 on `main`. If any of the four is missing, **you are not done with Day 1**.

## Cut-line (if time runs short)
- Skip Playwright entirely; stub a placeholder e2e job in CI. Keep vitest + lint + typecheck.
- Defer the e2e wiring to Day 2.

## Mistakes to avoid
- Writing `CLAUDE.md` as a human welcome doc. It is **operational orders for an agent**, not a README.
- Skipping branch protection. You will regret it by Day 5.
- Overcomplicating ESLint config — use a standard preset, move on.

## Validation signal — you understand this if...
...you can articulate, in one breath, why "ground truth before agents" is non-negotiable. If you feel the urge to skip to the fun agent stuff, reread the concept primer.

## References
- [Claude Code quickstart](https://docs.claude.com/en/docs/claude-code/quickstart)
- [CLAUDE.md / memory docs](https://docs.claude.com/en/docs/claude-code/memory)

## Today's artifact bundle
Committed to `main`:
- `package.json` with the five scripts.
- `.github/workflows/ci.yml` (from [06 §GitHub Actions starter](../06-artifacts.md#github-actions-starter)).
- `eslint` config, `vitest` config, `playwright.config.ts`.
- One unit spec, one e2e spec.
- `CLAUDE.md` v1 (from [06 §CLAUDE.md template](../06-artifacts.md#claudemd-template)).
- Branch protection rule on `main`.

Create `docs/agent-runs/day-1.md` with a 10-line note on what you did and what you'd redo.

Next: [Day 2 →](./day-2.md)
