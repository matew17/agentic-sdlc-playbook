# 10 — Alternative Task Sources (Linear, Jira, Asana)

> The control flow from [`02-target-architecture.md`](./02-target-architecture.md) stays exactly as designed. Only the **origin of the task** changes. This file is a drop-in swap for the "GitHub issue" node.

The design thesis: **the task source is an adapter**. Everything downstream — planner, PRD, branch, subagents, PR, gates — is unchanged. Swap the adapter and nothing else.

```
┌─────────────────┐       ┌──────────────────┐       ┌─────────────────────┐
│  Task source    │  -->  │  Adapter (you)    │  -->  │ Unchanged pipeline  │
│ Linear / Jira / │       │ webhook+fetcher   │       │ planner → ... → PR  │
│ Asana / GH      │       │                   │       │                     │
└─────────────────┘       └──────────────────┘       └─────────────────────┘
```

The adapter does three jobs:
1. **Trigger** — notify your system when a ticket is "ready for the agent."
2. **Fetch** — pull title, description, links, acceptance criteria, labels.
3. **Write-back** — update the ticket with PR link, status, comments.

That's it. Everything else is your existing architecture.

---

## Worked example: Linear

Linear is a good first target — it has a clean webhooks API, first-class labels, and decent markdown in descriptions. I recommend this pattern for your client if they use Linear.

### Moving parts

1. **A "ready-for-agent" label** in Linear (e.g., `agent:ready`). The trigger.
2. **A repo-level webhook receiver** — a GitHub Actions workflow triggered by `repository_dispatch`, fired from a tiny serverless function (or just a GitHub App you write).
3. **A Linear-aware slash command** in Claude Code: `/start-from-linear LIN-123` that does the fetch and kicks off the existing pipeline.
4. **Write-back step** that updates the Linear ticket with the PR URL and the run log link.

### Flow

```
Linear ticket gets label `agent:ready`
      │
      ▼
Linear webhook → Cloudflare Worker / Vercel fn / Lambda
      │  (HMAC-validated)
      ▼
repository_dispatch with payload { linearId, title, description }
      │
      ▼
.github/workflows/linear-dispatch.yml runs
      │  - checks out repo
      │  - runs claude code headlessly with /start-from-linear
      ▼
Planner fetches full ticket via Linear GraphQL, writes docs/prds/LIN-123-<slug>.md
      │
      ▼
  G1: human sets approved: true (or clicks Approve in Linear custom action)
      │
      ▼
Existing pipeline runs end-to-end → PR
      │
      ▼
Write-back: GitHub Action updates Linear ticket with PR URL, status, run log link
```

### Minimal code — the slash command

`.claude/commands/start-from-linear.md`:

```markdown
---
description: Start a feature from a Linear issue ID (e.g., ENG-123).
---

For Linear issue $ARGUMENTS:

1. Fetch the issue via Linear GraphQL:
   curl -s https://api.linear.app/graphql \
     -H "Authorization: $LINEAR_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"query":"query($id:String!){ issue(id:$id){ identifier title description labels{ nodes{ name } } comments{ nodes{ body } } } }","variables":{"id":"'"$ARGUMENTS"'"}}' > /tmp/issue.json

2. Construct a task brief from /tmp/issue.json using the Task Brief template (05-playbook.md). Persist as docs/tasks/$ARGUMENTS.md.

3. Invoke the `planner` subagent with that brief. PRD lands in docs/prds/$ARGUMENTS-<slug>.md with approved: false.

4. Wait for human approval (gate G1). Do not proceed until frontmatter says approved: true.

5. Create branch feat/$ARGUMENTS-<slug>. Call implementer → tester → reviewer as usual.

6. Open PR. Append to PR body:
   - Linear: https://linear.app/<workspace>/issue/$ARGUMENTS
   - Run log: docs/agent-runs/<date>-$ARGUMENTS.md

7. Post a comment back to Linear via GraphQL with the PR URL.
```

### The webhook → dispatch glue

A tiny Cloudflare Worker (or equivalent). Verifies HMAC, filters to label events, fires `repository_dispatch`:

```js
// worker.js — illustrative
export default {
  async fetch(req, env) {
    const body = await req.text();
    if (!verifyHmac(body, req.headers.get('Linear-Signature'), env.LINEAR_WEBHOOK_SECRET)) {
      return new Response('unauthorized', { status: 401 });
    }
    const evt = JSON.parse(body);
    const added = evt?.data?.labels?.filter(l => l.name === 'agent:ready');
    if (evt.type !== 'Issue' || evt.action !== 'update' || !added?.length) {
      return new Response('ignored');
    }
    await fetch(`https://api.github.com/repos/${env.OWNER}/${env.REPO}/dispatches`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${env.GH_TOKEN}`,
        'Accept': 'application/vnd.github+json',
      },
      body: JSON.stringify({
        event_type: 'linear_ready',
        client_payload: {
          identifier: evt.data.identifier,
          title: evt.data.title,
          description: evt.data.description,
          url: evt.url,
        },
      }),
    });
    return new Response('ok');
  },
};
```

### The GH workflow

`.github/workflows/linear-dispatch.yml`:

```yaml
name: linear-dispatch
on:
  repository_dispatch:
    types: [linear_ready]
permissions:
  contents: write
  pull-requests: write
jobs:
  start:
    runs-on: ubuntu-latest
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      LINEAR_API_KEY: ${{ secrets.LINEAR_API_KEY }}
      LIN_ID: ${{ github.event.client_payload.identifier }}
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - name: Kick off planner
        run: |
          npx -y @anthropic-ai/claude-code -p --agent planner \
            "Invoke /start-from-linear $LIN_ID. Produce PRD only; do not implement. Stop at G1."
      - name: Open PRD PR
        run: |
          BRANCH=prd/$LIN_ID
          git checkout -b $BRANCH
          git add docs/prds/ docs/tasks/
          git commit -m "prd($LIN_ID): draft via Linear dispatch"
          git push -u origin $BRANCH
          gh pr create --title "PRD: $LIN_ID (awaiting approval)" --body "Linear: ${{ github.event.client_payload.url }}"
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

### Write-back on merge

`.github/workflows/linear-writeback.yml` (triggers on PR closed/merged):

```yaml
name: linear-writeback
on:
  pull_request:
    types: [closed]
permissions: { pull-requests: read }
jobs:
  notify:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - name: Extract Linear ID from branch or PR body
        id: extract
        run: |
          ID=$(echo "${{ github.head_ref }}" | grep -oE '[A-Z]+-[0-9]+' | head -1)
          echo "id=$ID" >> $GITHUB_OUTPUT
      - name: Comment + close on Linear
        if: steps.extract.outputs.id != ''
        env: { LINEAR_API_KEY: ${{ secrets.LINEAR_API_KEY }} }
        run: |
          PR_URL="${{ github.event.pull_request.html_url }}"
          curl -s https://api.linear.app/graphql \
            -H "Authorization: $LINEAR_API_KEY" \
            -H "Content-Type: application/json" \
            -d "{\"query\":\"mutation(\$id:String!,\$body:String!){ commentCreate(input:{issueId:\$id, body:\$body}){ success } }\",\"variables\":{\"id\":\"${{ steps.extract.outputs.id }}\",\"body\":\"Merged via $PR_URL\"}}"
```

### Gotchas (in order of how often they bite)

1. **Label debounce.** A human adding multiple labels at once can fire multiple `repository_dispatch` events. Dedupe in the worker.
2. **Markdown drift.** Linear markdown ≠ GitHub markdown for some table/mention syntax. Have the planner normalize before writing the PRD.
3. **Comments as context.** Linear comments are often the *actual* spec. Fetch them in the GraphQL query; they're easy to miss.
4. **Write-back spam.** Don't comment on every PR state change — only on merge (and maybe on PR open). Reserve for human-visible events.
5. **ID format coupling.** Your branches (`feat/LIN-123-<slug>`) now encode the Linear ID. Document this in `CLAUDE.md` so future humans and agents understand the convention.

---

## Mapping to Jira

The shape is identical. Three swaps:

| Concern | Linear | Jira |
|---|---|---|
| Trigger | Label `agent:ready` | Transition to status `Ready for Agent` |
| Webhook | Linear webhooks (GraphQL) | Jira webhooks (REST) |
| Fetch | GraphQL query | `GET /rest/api/3/issue/{key}?expand=renderedFields` |
| Write-back | `commentCreate` mutation | `POST /rest/api/3/issue/{key}/comment` |
| ID format | `ENG-123` | `PROJ-123` |

Jira-specific notes:
- **Custom fields are ubiquitous.** You'll need to extract "Acceptance Criteria" from a custom field; these vary per org. Inspect with one issue's full JSON before writing the adapter.
- **ADF (Atlassian Document Format).** Jira returns descriptions as ADF, not markdown. Either ask for `renderedFields` (gives HTML) and post-process, or walk the ADF tree. I recommend `renderedFields` + a simple HTML→markdown convertor.
- **Workflow transitions are auditable.** Use them — `Ready for Agent → In Progress → In Review → Done` becomes your state machine, and Jira's audit log becomes free observability.

## Mapping to Asana

| Concern | Asana |
|---|---|
| Trigger | Custom field or tag change |
| Webhook | Asana webhooks |
| Fetch | `GET /tasks/{gid}` with `?opt_fields=notes,custom_fields,subtasks` |
| Write-back | `POST /tasks/{gid}/stories` (comment) |
| ID format | numeric gid — map to human slug in branch name |

Asana notes:
- Asana descriptions are HTML-ish. Normalize in the adapter.
- Subtasks make great checklist items for the planner; fetch them.
- Asana is often used by non-engineers — expect fuzzier acceptance criteria. Lean on the planner's PRD to pin them down.

## Mapping to GitHub Issues (status quo)

No adapter needed. Recap for completeness:

- Trigger: label `agent:ready` on an issue.
- Fetch: `gh issue view $N --json title,body,labels,comments`.
- Write-back: `gh issue comment $N`.

Included as the default in the [`/start-feature` slash command](./06-artifacts.md#slash-commands).

---

## When to *not* use an external source

- **Bug reports.** Keep bug triage in GitHub Issues even if features come from Linear. Bugs need tight loops with engineers; external task trackers add latency.
- **Spike / research tasks.** Don't shove exploratory work through the agent pipeline. It'll get through, but the PRD format is wrong for unstructured research.
- **Incident response.** Never. Humans drive incidents; agents assist post-incident with postmortems.

---

## Minimal contract any task source must satisfy

Steal this if your client uses something proprietary. Your adapter must provide:

1. A **stable identifier** (`LIN-123`).
2. A **title** (short, human-readable).
3. A **description** (ideally markdown; normalized if not).
4. **Acceptance criteria** (bulleted, testable) — either parsed from the description or a dedicated field.
5. **Links** (design, parent epic, related tickets) — captured as a list.
6. A **trigger signal** (label, status transition, tag).
7. A **write-back channel** (comment or field update).

If a system can provide these, you can plug it in without touching the rest of the architecture.

Back to the [top-level index](./README.md).
