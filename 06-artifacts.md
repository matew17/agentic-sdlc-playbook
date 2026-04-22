# 06 — Concrete Starter Artifacts

> Copy, adapt, commit. These are starters, not finished enterprise assets. The value is the *shape*.

## Contents

- [Repo / process structure](#repo--process-structure)
- [CLAUDE.md template](#claudemd-template)
- [Subagent — planner](#subagent-planner)
- [Subagent — implementer](#subagent-implementer)
- [Subagent — tester](#subagent-tester)
- [Subagent — reviewer](#subagent-reviewer)
- [Subagent — security](#subagent-security)
- [Skill example — `vue-component`](#skill-example--vue-component)
- [Slash commands](#slash-commands)
- [Hooks and settings](#hooks-and-settings)
- [Task decomposition template](#task-decomposition-template)
- [PRD / implementation brief template](#prd--implementation-brief-template)
- [Test strategy template](#test-strategy-template)
- [PR review checklist](#pr-review-checklist)
- [Definition of done checklist](#definition-of-done-checklist)
- [Agentic workflow decision matrix](#agentic-workflow-decision-matrix)
- [GitHub Actions starter](#github-actions-starter)
- [Run log template](#run-log-template)
- [Reusable prompts](#reusable-prompts)

---

## Repo / process structure

```
repo-root/
├── CLAUDE.md                         # agent standing orders (root)
├── README.md                         # human-facing
├── src/
│   ├── CLAUDE.md                     # optional: rules for src/ specifically
│   └── ...
├── tests/
├── e2e/
├── .claude/
│   ├── settings.json                 # permissions, hooks
│   ├── agents/
│   │   ├── planner.md
│   │   ├── implementer.md
│   │   ├── tester.md
│   │   ├── reviewer.md
│   │   └── security.md
│   ├── skills/
│   │   └── vue-component/
│   │       ├── SKILL.md
│   │       └── templates/
│   ├── commands/
│   │   ├── start-feature.md
│   │   ├── triage-ci.md
│   │   └── review-pr.md
│   └── hooks/
│       ├── block-sensitive-paths.sh
│       ├── typecheck-on-edit.sh
│       └── inject-context.sh
├── docs/
│   ├── adr/
│   │   └── 0001-agentic-sdlc-architecture.md
│   ├── prds/
│   │   └── 001-favorites.md
│   ├── agent-runs/
│   │   ├── TEMPLATE.md
│   │   └── YYYY-MM-DD-<slug>.md
│   └── agentic-intervention-policy.md
└── .github/
    └── workflows/
        ├── ci.yml
        ├── review.yml
        ├── triage.yml
        ├── dep-hygiene.yml
        └── deploy.yml
```

**Why this shape.** Everything agent-related is either in `.claude/` (runtime) or `docs/` (outputs). Your humans navigate `docs/`; your agents navigate `.claude/`. No mystery.

---

## CLAUDE.md template

```markdown
# Anime Catalog — Agent Operating Rules

## Stack
- Vue 3 + Vite + TypeScript
- pnpm (v9); Node 20
- Vitest for unit; Playwright for e2e; ESLint + @typescript-eslint

## Commands
- `pnpm install` — install
- `pnpm dev` — run dev server
- `pnpm typecheck` — tsc --noEmit
- `pnpm lint` — eslint
- `pnpm test` — vitest
- `pnpm test:e2e` — playwright
- `pnpm build` — vite build

## Architecture (5-line version)
- `src/views/` routes. `src/components/` presentational. `src/composables/` stateful logic.
- State is kept in composables + `localStorage`; no Pinia yet.
- Anime data is fetched from a public REST API; types in `src/types/anime.ts`.
- Do not add new top-level folders without an ADR.

## Agent rules (MUST)
- R1. Before writing code, call the `planner` subagent to produce `docs/prds/<id>-<slug>.md`. Human must approve by setting frontmatter `approved: true`.
- R2. For any `.vue` file touched, load skill `vue-component`.
- R3. Every task ends with a run log in `docs/agent-runs/`.
- R4. Tests MUST validate the PRD acceptance criteria, not the implementation.
- R5. Open PRs against `main` from `feat/<id>-<slug>` branches only.

## Agent rules (MUST NOT)
- R6. Do NOT push to `main` directly or merge PRs.
- R7. Do NOT edit `.github/workflows/`, `.claude/`, `docs/adr/`, or `.env*` without env var `CC_ALLOW_SENSITIVE=1`.
- R8. Do NOT add new dependencies without recording the reason in the run log.
- R9. Do NOT run commands beyond the allowlist in `.claude/settings.json`.

## Subagent routing
- requirements intake → `planner`
- code change → `implementer`
- tests → `tester`
- PR diff review → `reviewer`
- any diff touching auth/payments/crypto patterns → `security` (in addition)

## Human gates
- G1 PRD approval before implementation starts.
- G2 Human merges PR; CI + reviewer comment required.
- G3 Human approves production deploy in GitHub Environments.

## Cautious paths (human must co-edit)
- None currently. Add here when relevant.

## Run logs
- Location: `docs/agent-runs/YYYY-MM-DD-<slug>.md`
- Template: `docs/agent-runs/TEMPLATE.md`
- Always link the run log in the PR description.

## Style
- Prefer composables over in-component logic.
- Prefer `<script setup lang="ts">`.
- No default exports from components — named exports for testability.
```

---

## Subagent — planner

Path: `.claude/agents/planner.md`

```markdown
---
name: planner
description: Converts a task request into a PRD, task breakdown, and test strategy.
tools: Read, Write, Edit, Glob, Grep
model: claude-haiku-4-5
---

# Planner

## Role
Translate a task request into a reviewable plan. You do not write application code.

## Inputs
- A task brief (from the orchestrator or a GitHub issue).
- The repo `CLAUDE.md` and relevant source files discovered via Grep/Glob.

## Outputs
Exactly one file: `docs/prds/<id>-<slug>.md`, following the PRD template.
No code edits. No changes outside `docs/prds/`.

## Must-dos
1. Read `CLAUDE.md` first.
2. Discover relevant files before proposing design. Cite them by path.
3. Propose 2 implementation options when non-trivial; recommend one; explain why.
4. Enumerate acceptance criteria in testable language.
5. Propose a test strategy: which tests at which level (unit/e2e/manual) and why.
6. Include `approved: false` in frontmatter. The human flips it.

## Must-nots
- Do not write code.
- Do not decide architecture beyond the PRD scope; propose an ADR if the change is architectural.
- Do not assume context you have not read.

## Success
Human reads the PRD in < 5 minutes, understands the plan, and either approves or responds with specific changes.

## Escalation
If the task is ambiguous, end the PRD with a `## Questions` section. Do not invent answers.
```

---

## Subagent — implementer

Path: `.claude/agents/implementer.md`

```markdown
---
name: implementer
description: Writes application code to satisfy an approved PRD.
tools: Read, Write, Edit, Bash, Glob, Grep
model: claude-sonnet-4-6
---

# Implementer

## Role
Implement an approved PRD in the smallest diff that satisfies the acceptance criteria.

## Inputs
- An approved PRD (frontmatter `approved: true`).
- Repo source.

## Outputs
- Source code changes on the current feature branch.
- No changes to tests (the tester subagent handles those).
- No changes to `docs/` except appending to the run log.

## Must-dos
1. Read the PRD fully before editing.
2. Load skill `vue-component` if touching `.vue` files.
3. Write the smallest correct diff.
4. Run `pnpm typecheck` and `pnpm lint` after each edit batch; fix errors before proceeding.
5. Commit in small, narratively-titled commits (e.g., "feat(search): add debounce composable").

## Must-nots
- Do not add dependencies without justification in the run log.
- Do not refactor adjacent code "while you're there."
- Do not edit workflows, secrets, or `.claude/`.
- Do not write tests.

## Success
- `pnpm typecheck && pnpm lint` pass.
- Diff is scoped to the PRD.
- Code follows existing patterns.

## Escalation
If acceptance criteria conflict, stop and ask. If a dependency is missing, propose it in the run log and await approval.
```

---

## Subagent — tester

Path: `.claude/agents/tester.md`

```markdown
---
name: tester
description: Writes unit and e2e tests that validate PRD acceptance criteria.
tools: Read, Write, Edit, Bash, Glob, Grep
model: claude-sonnet-4-6
---

# Tester

## Role
Test **the spec**, not the implementation. Your job is to catch spec drift.

## Inputs
- Approved PRD.
- Optionally: implementation diff (but try to write tests from the PRD first).

## Outputs
- Unit tests in `tests/` or colocated `*.spec.ts`.
- E2E tests in `e2e/*.spec.ts`.
- No changes to `src/`.

## Must-dos
1. Map each PRD acceptance criterion to at least one test.
2. Write tests for error and empty states.
3. Write at least one e2e happy-path test if the feature has a UI.
4. Run `pnpm test && pnpm test:e2e`; ensure all pass.
5. In the run log, list criterion→test mapping.

## Must-nots
- Do not edit `src/` to make tests pass. If implementation is wrong, surface it.
- Do not write tests that pin the implementation's internal structure (unless that structure is part of the spec).

## Success
All PRD acceptance criteria have associated tests; suite is green.

## Escalation
If a criterion cannot be tested automatically, call it out and propose a manual test step.
```

---

## Subagent — reviewer

Path: `.claude/agents/reviewer.md`

```markdown
---
name: reviewer
description: Reviews diffs as a skeptical senior engineer against PRD + checklist.
tools: Read, Bash, Glob, Grep
model: claude-haiku-4-5
---

# Reviewer

## Role
Find what the implementer and tester missed. Bias toward skepticism.

## Inputs
- The PR diff (via `git diff main...HEAD`).
- The PRD file referenced in the run log.
- `CLAUDE.md`.

## Outputs
- A single markdown comment body posted to the PR, with sections: Summary, Blocking, Suggestions, Nits.
- No file edits. No approvals (humans merge).

## Must-dos
1. Run the full PR review checklist (see `docs/playbook/pr-review-checklist.md`).
2. Check accessibility for UI changes (ARIA, keyboard, focus, color contrast).
3. Check error/loading/empty states.
4. Check for scope creep against the PRD.
5. Distinguish Blocking (merge-stoppers) from Suggestions.

## Must-nots
- Do not edit files.
- Do not approve or merge.
- Do not suggest changes not grounded in the diff, PRD, or CLAUDE.md rules.

## Success
Review is specific, actionable, and terse. Max 600 words.
```

---

## Subagent — security

Path: `.claude/agents/security.md`

```markdown
---
name: security
description: Scans diffs for security issues before merge.
tools: Read, Bash, Glob, Grep
model: claude-sonnet-4-6
---

# Security

## Role
Flag potential security issues in a diff. You are not a full audit; you are a gate.

## Inputs
- PR diff.
- Repo source for context.

## Outputs
- Structured comment: CRITICAL / HIGH / MEDIUM / LOW / INFO, one bullet each.
- If CRITICAL or HIGH, emit a "REQUEST CHANGES"-style comment.

## Must-check
- Secrets committed (even in tests, fixtures, comments).
- XSS / injection patterns (`v-html`, template interpolation into attrs, dangerouslySetInnerHTML analogs, SQL/shell string interp).
- Auth/authz changes (middleware, route guards, role checks).
- Cryptography use (random, hashing, signing, jwt).
- External input handled without validation.
- `eval`, dynamic `require`/`import`, unchecked JSON parsing.
- Dependency additions (check for known-bad packages).

## Must-nots
- Do not fix issues yourself. File issues for the implementer to fix.
```

---

## Skill example — `vue-component`

Path: `.claude/skills/vue-component/SKILL.md`

```markdown
---
name: vue-component
description: Authoring rules and templates for Vue 3 SFCs in this repo. Load when editing .vue files.
---

# vue-component

Follow these when creating or editing a Vue SFC.

## Structure
```vue
<script setup lang="ts">
import { computed, ref } from 'vue'

interface Props { title: string; count?: number }
const props = withDefaults(defineProps<Props>(), { count: 0 })
const emit = defineEmits<{ (e: 'select', id: string): void }>()

const doubled = computed(() => props.count * 2)
</script>

<template>
  <section :aria-label="props.title">
    <!-- content -->
  </section>
</template>

<style scoped>
/* nothing global */
</style>
```

## Rules
- Always `<script setup lang="ts">`.
- Props via `defineProps<Interface>()`. No prop defaults via runtime objects except via `withDefaults`.
- Emit types required.
- Accessibility: always provide `aria-label` on top-level regions; use semantic HTML.
- No global styles; `scoped` only.
- No direct localStorage/sessionStorage in templates — use a composable.

## Tests
- Colocated `Component.spec.ts` using `@vue/test-utils`.
- Render with required props; assert on output and emitted events.

## Anti-patterns
- `v-html` with any non-trusted data.
- Reaching into child components via `$refs` for state.
- Business logic in `<script setup>` longer than ~60 lines — extract to a composable.
```

---

## Slash commands

Path: `.claude/commands/start-feature.md`

```markdown
---
description: Start a feature from a GitHub issue number.
---

For issue $ARGUMENTS:

1. Use the GitHub CLI to fetch the issue: `gh issue view $ARGUMENTS --json title,body,labels`.
2. Invoke the `planner` subagent with the issue title/body as the task brief.
3. When the PRD exists and has `approved: true`, create a branch `feat/$ARGUMENTS-<slug>`.
4. Invoke `implementer` with the PRD.
5. Invoke `tester` with the PRD and the implementer's diff.
6. Invoke `reviewer` on the branch.
7. If reviewer Blocking is empty, push and open a PR using `gh pr create`, body generated from the run log.
8. Write/update `docs/agent-runs/<date>-<slug>.md`.
```

Path: `.claude/commands/triage-ci.md`

```markdown
---
description: Summarize a failing CI run and propose the fix.
---

1. `gh run view $ARGUMENTS --log-failed | head -n 200`.
2. Identify the failing job and the first error.
3. Hypothesize 2 root causes; pick the most likely and why.
4. Propose a minimal patch as a suggestion; do not apply.
5. Post a PR comment (if on a PR) summarizing findings in <= 200 words.
```

Path: `.claude/commands/review-pr.md`

```markdown
---
description: Run the reviewer subagent on a PR.
---

1. `gh pr diff $ARGUMENTS > /tmp/pr.diff`.
2. Identify the PRD reference from the PR body.
3. Invoke the `reviewer` subagent with the diff and PRD.
4. Post the reviewer's output as a PR comment via `gh pr comment`.
```

---

## Hooks and settings

Path: `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Read(*)",
      "Write(**)",
      "Edit(**)",
      "Glob(*)",
      "Grep(*)",
      "Bash(pnpm install)",
      "Bash(pnpm run *)",
      "Bash(pnpm test*)",
      "Bash(pnpm lint*)",
      "Bash(pnpm typecheck*)",
      "Bash(pnpm build*)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git add *)",
      "Bash(git commit -m *)",
      "Bash(git checkout -b feat/*)",
      "Bash(git push origin feat/*)",
      "Bash(gh issue view *)",
      "Bash(gh pr create *)",
      "Bash(gh pr comment *)",
      "Bash(gh pr diff *)",
      "Bash(gh run view *)"
    ],
    "deny": [
      "Bash(git push origin main)",
      "Bash(git push --force*)",
      "Bash(git merge *)",
      "Bash(gh pr merge *)",
      "Bash(rm -rf *)",
      "Edit(.env*)",
      "Edit(.github/workflows/**)",
      "Edit(docs/adr/**)"
    ]
  },
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "command": ".claude/hooks/block-sensitive-paths.sh"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "command": ".claude/hooks/typecheck-on-edit.sh"
      }
    ],
    "UserPromptSubmit": [
      { "command": ".claude/hooks/inject-context.sh" }
    ]
  }
}
```

Path: `.claude/hooks/block-sensitive-paths.sh`

```bash
#!/usr/bin/env bash
# Reads tool input from stdin, blocks edits in sensitive paths unless explicitly allowed.
set -euo pipefail

INPUT=$(cat)
PATH_EDITED=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

if [ -z "$PATH_EDITED" ]; then exit 0; fi

case "$PATH_EDITED" in
  *auth/*|*payments/*|*db/migrations/*|.github/workflows/*|*.env*)
    if [ "${CC_ALLOW_SENSITIVE:-0}" != "1" ]; then
      echo '{"decision":"block","reason":"Sensitive path. Set CC_ALLOW_SENSITIVE=1 and confirm with human before editing."}'
      exit 0
    fi
    ;;
esac
exit 0
```

Path: `.claude/hooks/typecheck-on-edit.sh`

```bash
#!/usr/bin/env bash
# Runs typecheck after any edit touching TS/Vue files; adds result to context.
set -euo pipefail
INPUT=$(cat)
PATH_EDITED=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

case "$PATH_EDITED" in
  *.ts|*.tsx|*.vue)
    OUTPUT=$(pnpm -s typecheck 2>&1 || true)
    echo "{\"additional_context\": $(jq -Rs . <<< "$OUTPUT")}"
    ;;
esac
exit 0
```

Make both executable: `chmod +x .claude/hooks/*.sh`.

---

## Task decomposition template

Path: `docs/templates/task-decomposition.md`

```markdown
# Task: <title>

## Summary
<1–2 lines>

## Acceptance criteria
- AC1 …
- AC2 …

## Decomposition
| # | Step | Owner | Ground truth | Reversibility |
|---|------|-------|--------------|---------------|
| 1 | Create PRD | planner | human approval | revert file |
| 2 | Scaffold types/composable | implementer | tsc | revert commit |
| 3 | UI component | implementer | vitest + manual | revert commit |
| 4 | Wiring & route | implementer | e2e | revert commit |
| 5 | Tests | tester | test suite | revert commit |
| 6 | Review | reviewer | checklist | comments only |
| 7 | PR + CI | orchestrator | required checks | PR closed |

## Risks & unknowns
- …

## Out of scope
- …
```

---

## PRD / implementation brief template

Path: `docs/prds/TEMPLATE.md`

```markdown
---
id: <id>
title: <title>
approved: false
approver: <name>
created: <YYYY-MM-DD>
---

# PRD: <title>

## Context
Why this change. Link to issue / Slack / ADR if any.

## Users and scenarios
Who benefits, when do they hit this flow.

## Acceptance criteria
- AC1: testable statement.
- AC2: …

## Non-functional
- Performance budget (e.g., <16ms interaction).
- Accessibility (keyboard, SR, contrast).
- Privacy / data handling.

## Options considered
1. **Option A — <name>.** Pros / cons.
2. **Option B — <name>.** Pros / cons.

**Recommendation:** Option X because …

## Proposed design
Files to add/change. Data flow. Edge cases.

## Test strategy
Unit: …
E2E: …
Manual: …

## Out of scope
- …

## Risks
- …

## Questions
- …  (human answers before approval)
```

---

## Test strategy template

Path: `docs/templates/test-strategy.md`

```markdown
# Test strategy: <feature>

## Principle
Tests validate the PRD, not the implementation.

## Mapping
| AC | Test type | Test name | Status |
|----|-----------|-----------|--------|
| AC1 | unit | `filter.spec.ts:it('filters by title, case-insensitive')` | ☐ |
| AC2 | e2e | `e2e/search.spec.ts` | ☐ |

## Coverage targets
- Logic (composables, utils): 90%+ lines, 85%+ branches.
- Components: behavioral — render + interactions. No snapshot tests.
- E2E: one happy path, one failure path per feature.

## Explicitly not tested
- …

## Manual QA
- Keyboard navigation.
- Screen reader labels read correctly.
- Dark mode visuals.
```

---

## PR review checklist

Path: `docs/templates/pr-review-checklist.md`

```markdown
# PR Review Checklist

## Scope & spec
- [ ] Diff matches PRD scope; no unrelated changes.
- [ ] All PRD acceptance criteria are satisfied.
- [ ] Out-of-scope items listed in PRD are honored.

## Correctness
- [ ] Types pass (no `any` introduced without comment).
- [ ] Lint passes; no disabled rules.
- [ ] Error states handled.
- [ ] Empty / loading states handled.
- [ ] Race conditions considered (debounce, cancel, stale responses).

## Tests
- [ ] Unit tests cover new logic.
- [ ] E2E added if user-facing.
- [ ] Tests fail without the change (confirmed mentally or by a flip).
- [ ] No tests assert on irrelevant internal structure.

## UX / frontend
- [ ] Accessible: labels, roles, keyboard path, focus management.
- [ ] Responsive: works on narrow viewport.
- [ ] i18n-safe: no hardcoded strings in a translated app.
- [ ] No console errors/warnings in dev server.

## Security
- [ ] No secrets in diff.
- [ ] No new `v-html` / dangerouslySetInnerHTML.
- [ ] New deps justified; no known-malicious package names.

## Observability
- [ ] Logs meaningful; no `console.log` left in production paths.
- [ ] Errors surfaced via existing error channel.

## Agent-specific
- [ ] Run log committed under `docs/agent-runs/` and linked in PR body.
- [ ] PR body generated from run log narrates decisions.
- [ ] Reviewer subagent comment is present and addressed.
```

---

## Definition of done checklist

Path: `docs/templates/definition-of-done.md`

```markdown
# Definition of Done

A change is DONE when ALL of the following are true:

- [ ] PRD approved (`approved: true`).
- [ ] All PRD acceptance criteria satisfied.
- [ ] Unit + e2e tests added and passing.
- [ ] Typecheck, lint, build all green locally and in CI.
- [ ] Reviewer subagent comment present; all Blocking items resolved.
- [ ] Security subagent comment present; no HIGH/CRITICAL open.
- [ ] Human reviewer has approved the PR.
- [ ] Run log committed and linked from PR body.
- [ ] PR title follows conventional commits (`feat|fix|chore|refactor(scope): ...`).
- [ ] No new dependencies without justification.
- [ ] No `TODO` / `FIXME` without linked issue.
- [ ] Deploy smoke check passes (for feature-flagged or stage-deployed changes).
```

---

## Agentic workflow decision matrix

Path: `docs/templates/decision-matrix.md`

```markdown
# Agentic workflow decision matrix

Score each proposed automation 1–5. Sum = posture.

| Dimension | 1 | 3 | 5 |
|---|---|---|---|
| Reversibility | Irreversible | Hours to revert | One-commit revert |
| Blast radius | All users | Team | Single file |
| Ground truth | None | Partial | Strong tests + types |
| Repeatability | Novel | Familiar | Routine |
| Sensitivity | Auth/pay/crypto | Business logic | Docs/UI |

**Posture by total score**
- 20–25: fully autonomous; human approves merge only.
- 13–19: agent implements, human reviews PR.
- 7–12: cautious mode; human co-writes each change.
- ≤ 6: human leads; agent assists (planner/reviewer only).

**Quick veto list (always human-led regardless of score)**
- Auth / authz.
- Payments / financial calculations.
- Schema migrations / data backfills.
- Production IaC (IAM, network, storage).
- Public API breaking changes.
- Cryptography.
```

---

## GitHub Actions starter

Path: `.github/workflows/ci.yml`

```yaml
name: ci
on:
  pull_request:
  push: { branches: [main] }
permissions:
  contents: read
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm typecheck
      - run: pnpm lint
      - run: pnpm test -- --coverage
      - run: pnpm build
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps
      - run: pnpm test:e2e
```

Path: `.github/workflows/review.yml` (agent PR reviewer)

```yaml
name: agent-review
on:
  pull_request:
    types: [opened, synchronize, reopened]
permissions:
  contents: read
  pull-requests: write
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Compute diff
        run: |
          git fetch origin ${{ github.base_ref }}
          git diff origin/${{ github.base_ref }}...HEAD > /tmp/pr.diff
          echo "PR_NUM=${{ github.event.pull_request.number }}" >> $GITHUB_ENV
      - name: Run reviewer subagent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # Headless Claude Code with reviewer subagent as the system prompt.
          # The reviewer reads CLAUDE.md, the PRD referenced in the PR body, and /tmp/pr.diff.
          npx -y @anthropic-ai/claude-code \
            -p --agent reviewer \
            "$(cat <<'EOF'
          Review the diff at /tmp/pr.diff against CLAUDE.md and the PRD linked in the PR body.
          Post output in the shape:
          ## Summary
          ## Blocking
          ## Suggestions
          ## Nits
          EOF
          )" > /tmp/review.md
      - name: Post review comment
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: gh pr comment "$PR_NUM" --body-file /tmp/review.md
```

Path: `.github/workflows/triage.yml` (CI failure triage)

```yaml
name: agent-triage
on:
  workflow_run:
    workflows: [ci]
    types: [completed]
permissions:
  contents: read
  pull-requests: write
  actions: read
jobs:
  triage:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Fetch failing logs
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: gh run view ${{ github.event.workflow_run.id }} --log-failed | head -n 400 > /tmp/logs.txt
      - name: Summarize
        env: { ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }} }
        run: |
          npx -y @anthropic-ai/claude-code -p \
            "Read /tmp/logs.txt. Identify the failing job, the first error, two likely root causes, and a proposed minimal fix. <=200 words." > /tmp/triage.md
      - name: Comment on associated PR
        if: ${{ github.event.workflow_run.event == 'pull_request' }}
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          PR_NUM=$(gh pr list --search "${{ github.event.workflow_run.head_sha }}" --json number --jq '.[0].number')
          gh pr comment "$PR_NUM" --body-file /tmp/triage.md
```

Path: `.github/workflows/dep-hygiene.yml` (weekly dep PR)

```yaml
name: dep-hygiene
on:
  schedule: [{ cron: '0 8 * * 1' }]
  workflow_dispatch:
permissions:
  contents: write
  pull-requests: write
jobs:
  bump:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - name: Compute updates
        run: pnpm outdated --format json > /tmp/outdated.json || true
      - name: Agent-guided patch/minor bumps
        env: { ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }} }
        run: |
          npx -y @anthropic-ai/claude-code -p \
            "Read /tmp/outdated.json. Propose pnpm commands to bump only patch + minor versions (no majors). Output only bash commands, one per line." > /tmp/cmds.sh
          bash /tmp/cmds.sh || true
          pnpm install
      - name: Open PR if any changes
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          if ! git diff --quiet; then
            git checkout -b chore/deps-$(date +%Y%m%d)
            git add -A
            git commit -m "chore(deps): weekly patch/minor bumps"
            git push -u origin HEAD
            gh pr create --title "chore(deps): weekly patch/minor bumps" --body "Automated. Majors excluded."
          fi
```

Path: `.github/workflows/deploy.yml`

```yaml
name: deploy
on:
  push: { branches: [main] }
  workflow_dispatch:
permissions:
  contents: read
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "deploy to staging"   # replace with your actual deploy
  deploy-production:
    environment: production  # required reviewer configured in GitHub Environments
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "deploy to production" # gated by human approval
```

---

## Run log template

Path: `docs/agent-runs/TEMPLATE.md`

```markdown
---
task: <id-slug>
date: <YYYY-MM-DD>
branch: feat/<id>-<slug>
pr: <#>
agents_used: [planner, implementer, tester, reviewer]
models: { planner: haiku, implementer: sonnet, tester: sonnet, reviewer: haiku }
---

# Run log — <title>

## Brief
<one line>

## PRD
[docs/prds/<id>-<slug>.md](../prds/<id>-<slug>.md)

## Decisions
- D1: …
- D2: …

## Subagent calls
1. planner — input <summary>; output <path>; duration <m:ss>; cost <est>.
2. implementer — …
3. tester — …
4. reviewer — …

## Diffs of note
- <file> — <why>

## Failures and retries
- <what failed, what we changed, what fixed it>

## Open tech debt
- …

## Follow-ups
- …
```

---

## Reusable prompts

Drop in PR body, issue, or orchestrator.

**"Kick off a feature"**
```
You are the orchestrator. Task: <one line>. Follow the playbook in CLAUDE.md:
1) Invoke planner to produce docs/prds/<id>-<slug>.md. Stop and wait for human approval.
2) When approved, create branch feat/<id>-<slug>.
3) Invoke implementer with the PRD.
4) Invoke tester with the PRD + implementer diff.
5) Invoke reviewer; resolve any Blocking items.
6) Push branch, open PR, paste run log as body.
If stuck at any step, write a `## Questions` block and stop.
```

**"Fix a weak PR"**
```
Do not rewrite. Surgical repair:
- Missing tests for acceptance criteria <list>. Tester subagent, add them.
- File <path> is out of scope vs PRD. Revert it.
- Error state for <scenario> not handled. Implementer, add it.
Rerun reviewer when done.
```

**"Audit a sensitive change"**
```
Security subagent, review the diff for: secrets, XSS, auth/authz, crypto misuse,
unchecked external input, injection, new deps. Classify findings as CRITICAL/HIGH/
MEDIUM/LOW/INFO. If CRITICAL or HIGH, emit REQUEST CHANGES.
```

**"Write an ADR"**
```
We are deciding: <question>. Options: <A, B, C>. Write docs/adr/NNNN-<slug>.md with:
context, considered options, decision, consequences, and a rollback plan. Leave
`status: proposed` until a human accepts it.
```

**"Triage a flaky test"**
```
Test <name> failed <N>x in the last week. Read history in git log and CI logs.
Classify as: genuine bug / race / environment / timing. Propose either (a) a
deterministic fix or (b) quarantine with a tracking issue. Do not delete the test.
```

Next: [07-failure-modes.md](./07-failure-modes.md).
