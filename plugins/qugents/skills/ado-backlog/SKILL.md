---
name: ado-backlog
description: Conventions for running the QuGents agent team from an Azure DevOps (Scrum) backlog - work item types and fields, the agent tag workflow (agent-plan, agent-proposed, agent-ready, agent-working, agent-pr-open, needs-human), WIQL queries, branch/commit/PR naming and the Azure DevOps MCP tools. Loaded by ado-plan and ado-deliver; use directly when reading or updating backlog items for the agents.
---

# Azure DevOps backlog for the agent team

The product owner (PO) works in Azure DevOps Boards. The agents read the backlog, plan, build and open pull requests. **Only the PO decides what gets built and what gets merged.**

## Project settings

Read the `## Azure DevOps` section of the repo's `CLAUDE.md`:

```markdown
## Azure DevOps
- Organization: contoso
- Project: Pos
- Team: Pos Team
- Repository: Pos
- Default branch: main
```

If the section is missing, ask the user once for these values and add the section to `CLAUDE.md`.

## Tools

Azure DevOps is reached through the local **Azure DevOps MCP server** (`@azure-devops/mcp`, domains `core work work-items repositories`). Its tools appear as `mcp__azure-devops__…`. The ones used here:

| Need | Tool / action |
|---|---|
| Read one / several items | `wit_work_item` → `get`, `get_batch` |
| Find items | `wit_query` → `wiql` (pass `project` and `team`) |
| Read comments | `wit_work_item` → `list_comments` |
| Create child tasks | `wit_work_item_write` → `add_child` (`parentId`, `workItemType: "Task"`, `items: [{title, description, format: "Markdown"}]`). It cannot set tags or other fields |
| Update fields, tags, state | `wit_work_item_write` → `update` (`updates: [{op: "replace", path: "/fields/System.Tags", value: "…"}]`, prepend `{op: "test", path: "/rev", value: <rev>}` to avoid overwriting a concurrent edit) or `update_batch` for several items |
| Comment | `wit_work_item_comment_write` → `add` |
| Dependencies | `wit_work_item_link_write` → `link` (`updates: [{id, linkToId, type: "predecessor"}]`) |
| Pull request | `repo_pull_request_write` → `create` (`repositoryId` + `project`, `sourceRefName: "refs/heads/…"`, `targetRefName`, `workItems: "<taskId>"` links the task; `description` max 4000 characters) |

If the MCP tools are not available, stop and show the setup command from the README (`claude mcp add … @azure-devops/mcp …`). Never fall back to guessing work item content.

Git (branches, commits, push) is done with the local `git` CLI, using the user's credentials.

## Scrum process mapping

| Level | Work item type | Who writes it |
|---|---|---|
| Goal | Epic / Feature | PO |
| What to build | **Product Backlog Item (PBI)**: Description + **Acceptance Criteria** | PO |
| How to build it | **Task** (child of the PBI): one vertical slice, one pull request | planner (agents) |

Fields used:

| Field | Reference name |
|---|---|
| Title | `System.Title` |
| Description (Markdown or HTML) | `System.Description` |
| Acceptance criteria (PBI only) | `Microsoft.VSTS.Common.AcceptanceCriteria` |
| Tags (`;`-separated) | `System.Tags` |
| State | `System.State` (PBI: New → Approved → Committed → Done; Task: To Do → In Progress → Done) |
| Order | `Microsoft.VSTS.Common.BacklogPriority` (PBI), Task title prefix `T1`, `T2`… |
| Remaining work (Task) | `Microsoft.VSTS.Scheduling.RemainingWork` (S=2, M=4, L=8) |

Tasks have no acceptance-criteria field, so their criteria go in the Description.

## Tag workflow

| Tag | Set by | Meaning |
|---|---|---|
| `agent-plan` | PO, on a PBI | Please break this PBI into tasks |
| `agent-planned` | agent, on the PBI | Tasks were created (replaces `agent-plan`) |
| `agent-proposed` | agent, on new Tasks | Waiting for PO review |
| `agent-ready` | **PO**, on a Task | Approved for building. The only tag that lets an agent write code |
| `agent-working` | agent | Claimed; a team run is in progress |
| `agent-pr-open` | agent | Pull request open, waiting for PO review/merge |
| `needs-human` | agent | Blocked: question, failed build, or failed review. See the latest comment |

Rules:
- Change tags by editing `System.Tags` as a whole: read the current value, add/remove tags, write it back. Keep tags the PO added.
- Never add `agent-ready` yourself, never close or delete work items, never change PBI state or fields other than tags.
- Every agent action on a work item gets a short comment saying what happened and what the PO should do next.

## WIQL queries

PBIs to plan:

```sql
SELECT [System.Id] FROM WorkItems
WHERE [System.TeamProject] = @project
  AND [System.WorkItemType] = 'Product Backlog Item'
  AND [System.Tags] CONTAINS 'agent-plan'
  AND [System.State] <> 'Done'
ORDER BY [Microsoft.VSTS.Common.BacklogPriority] ASC
```

Tasks ready to build:

```sql
SELECT [System.Id] FROM WorkItems
WHERE [System.TeamProject] = @project
  AND [System.WorkItemType] = 'Task'
  AND [System.Tags] CONTAINS 'agent-ready'
  AND [System.Tags] NOT CONTAINS 'agent-working'
  AND [System.State] = 'To Do'
ORDER BY [System.Id] ASC
```

Pick order among ready tasks: the parent PBI's `BacklogPriority` first, then the `T<n>` number in the title. Skip a task whose predecessor tasks are not Done (or not at least `agent-pr-open` if the PO has said stacking is fine).

## Git and pull request conventions

- Branch: `agent/<taskId>-<short-slug>` from an up-to-date default branch (`git switch main && git pull`).
- Commits: imperative subject, then a line `AB#<taskId>` so Azure Repos links the commit.
- Never commit to or push the default branch. Never force-push someone else's branch.
- PR title: `AB#<taskId> <task title>`. Target: the default branch. Description:

```markdown
## What
<2–4 bullets from the developer hand-off>

## How to test
<steps; URLs https://localhost:5001 / https://localhost:4200>

## Tests
<tester report: added tests, result>

## Review
<reviewer verdict + remaining non-blocking findings>

AB#<taskId>
```

- The task is linked through `workItems` on create (and `AB#<id>` in the description).
- The agents never approve, vote on, set auto-complete, bypass policies on, complete or merge pull requests (`vote`, `autoComplete`, `bypassPolicy` are off-limits).
