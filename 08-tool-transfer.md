# 08 — Transferring the Approach from Claude Code to Other Tools

> The architecture and playbook are the asset. The runtime (Claude Code) is replaceable. Here is how each concept maps.

---

## The invariants (these survive any runtime change)

These are your asset regardless of tool:
- The **orchestrator + specialists** shape.
- The **authority model** (who may do what).
- **Git + PR + CI** as the control plane.
- The **three human gates**.
- **File-based memory**: PRDs, ADRs, run logs.
- **Decision matrix** and **intervention policy**.
- **Subagent design contract** (7-question template).
- **Review ritual**.

If you do nothing else, write these down in a stack-independent doc and carry them.

---

## Concept mapping table

| Concept | Claude Code | Devin | Cursor Agent / Windsurf | OpenAI Codex / Custom (SDK + Python) |
|---|---|---|---|---|
| Project memory | `CLAUDE.md`, nested CLAUDE.md | Knowledge base entries; playbooks | `.cursorrules` / memory | System prompt + loaded files |
| Subagent | `.claude/agents/*.md` | A "playbook" + scoped Devin session | Agent mode with rules file | Multiple model calls with distinct system prompts |
| Skill | `.claude/skills/*/SKILL.md` | Playbook with attached scripts | Rule file + snippets | Tool manifest + retrieved doc |
| Slash command | `.claude/commands/*.md` | Saved task template | Cursor command | A function in your harness |
| Hook (pre/post tool) | `.claude/hooks/*.sh` | Sandbox VM constraints | Limited; rules only | Your own pre/post middleware |
| Permissions | `.claude/settings.json` allow/deny | Sandbox + ACLs | Limited | Your own allowlist |
| Tool use | Bash, Edit, Read | Bash, Editor in VM | Bash, Edit | MCP or custom tool dispatch |
| Human gate | Prompt to approve + branch rules | Ask-follow-up + PR | Prompt + PR | Your own UI/flow |
| Run log | `docs/agent-runs/` (you commit) | Devin's run history + your commit | You commit | You commit |
| CI integration | Claude Code Action / `-p` headless | Devin CI tasks | Headless runs | SDK in Actions |

---

## Mapping to Devin specifically

Devin is a managed, opinionated agent: a VM, an inbox, and a full-task runtime. It's strong at **going away and doing a task** but weaker at the fine-grained orchestration this playbook depends on. Map as follows:

### What transfers straight
- **Task briefs** — use the same template from [`05-playbook.md`](./05-playbook.md#task-brief-pattern-prompt-template). Devin's input field wants exactly this shape.
- **PRDs and ADRs** — Devin reads from your repo; commit them the same way.
- **Human gates** — G1 (PRD approval) enforced by your workflow outside Devin; G2 via GitHub branch protection; G3 via GH Environments. Unchanged.
- **Authority model** — Devin runs in its own sandbox, but you still control *what it can merge or deploy* via GitHub rules. That's what matters.

### What changes
- **Subagents** — Devin is one agent per session. To get specialist behavior, run **separate Devin sessions** per role (planner session, implementer session, reviewer session) with different task briefs. Less elegant; same shape.
- **Skills** — Devin has "playbooks" (reusable task templates). Keep your skill content, but store it both as `SKILL.md` in the repo (for portability) *and* as a Devin playbook.
- **Hooks** — Devin has no PreToolUse hook. Put enforcement in CI (required checks, branch protection, `gitleaks`), not in the agent runtime. This is actually healthier anyway.
- **Permissions** — Devin's VM is sandboxed; the *real* permission boundary shifts to your GitHub App installation scope and branch rules. Lock those down.

### Division of labor recommendation
If the client uses both: let **Claude Code** handle the tight-feedback-loop work (feature dev, iteration, PR review assistant) and let **Devin** handle the fire-and-forget tasks (dep upgrades, large refactors that take hours, multi-PR sweeps). The strengths complement.

### Cost / control tradeoffs
- Devin: lower per-task orchestration effort; higher $/task; harder to debug.
- Claude Code: more setup; lower $/task; fully inspectable.
- If you're doing 100+ routine tasks/month, Devin-for-routine + Claude-Code-for-features is the pragmatic mix.

---

## Mapping to Cursor Agent / Windsurf Cascade

- Best for **individual-engineer dev-loop assistance**, not SDLC orchestration.
- Use `.cursorrules` as your `CLAUDE.md`-equivalent.
- Keep your subagents in repo as markdown; even if the tool doesn't natively consume them, they remain documentation and portable assets.
- Rely on GH-Actions-based agents (you can still use Claude Code headless) for the CI review/triage loop regardless of what editor your team uses.

---

## Mapping to a custom harness (Anthropic SDK + Python/TS)

Use this if compliance prohibits CLI agents in CI, or you need custom observability.

- **One Python/TS service** that dispatches to Claude via the SDK.
- Each "subagent" is a function with its own system prompt + tool list.
- Log every call to a structured store; emit the same markdown run log.
- Trigger via GH webhooks → API Gateway → service.
- **Resist the urge to build a framework.** Keep it dumb: a router and five functions. The playbook shape does the work, not the code.

---

## Portability checklist

When you hand off or migrate, ensure these travel:

- [ ] `CLAUDE.md` (and any nested ones).
- [ ] `.claude/agents/*.md` — subagent specs (markdown is portable).
- [ ] `.claude/skills/**` — skills with templates.
- [ ] `.claude/commands/*.md` — slash command specs (re-implement per tool).
- [ ] `.claude/settings.json` — permissions ruleset (translate to target tool).
- [ ] `docs/adr/*`, `docs/prds/*`, `docs/agent-runs/*` — history.
- [ ] `docs/agentic-intervention-policy.md` — the politically important doc.
- [ ] `.github/workflows/*.yml` — mostly portable; adjust the agent call line.
- [ ] Branch protection rules — export/document.
- [ ] Required secrets list (never the values).

---

## Decision: which tool, when

Opinionated defaults:

- **Solo / small team, front-end-heavy, senior operator** → Claude Code. Best per-dollar control.
- **Enterprise, heavy compliance, wants "click-to-run" UX** → Devin for routine, Claude Code (or SDK-in-CI) for bespoke.
- **Mixed team, low AI fluency, need editor UX** → Cursor Agent for IDE; Claude Code headless in CI.
- **Regulated fintech (likely your client)** → Claude Code or SDK-in-CI with strict allowlists; evaluate Devin for non-regulated routine only.

Your client is fintech + HR. Start with Claude Code inside the SDLC plus SDK-in-CI for headless review/triage. Add Devin only after a security review for the specific task types you'd delegate to it.

Next: [09-what-good-looks-like.md](./09-what-good-looks-like.md).
