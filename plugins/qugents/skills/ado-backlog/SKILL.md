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
- Code host: GitHub                                   (or: Azure Repos)
- Backend repository: GitQuuu/Pos (local: .)
- Frontend repository: GitQuuu/Pos-web (local: ../Pos-web)
- Integration branch: development
```

Backend (.NET API) and frontend (Angular) are **separate repositories**. `local:` is the folder relative to the repo whose `CLAUDE.md` you are reading; put the same section in both repos' `CLAUDE.md` with paths relative to each. Start Claude Code in the backend repo and give it access to the frontend folder (`claude --add-dir ../Pos-web`, or `"permissions": {"additionalDirectories": ["../Pos-web"]}` in `.claude/settings.json`). A project with only one repository uses `- Repository: <owner>/<repo>` instead.

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
| Pull request, **Azure Repos** | `repo_pull_request_write` → `create` (`repositoryId` + `project`, `sourceRefName: "refs/heads/…"`, `targetRefName`, `workItems: "<taskId>"` links the task; `description` max 4000 characters) |
| Pull request, **GitHub** | `gh pr create --base development --head agent/<id>-<slug> --title … --body-file …` (GitHub CLI, user's login). The Azure Boards GitHub app links `AB#<id>` and closes the task on merge with `Fixes AB#<id>` |

If the MCP tools are not available, stop and show the setup command from the README (`claude mcp add … @azure-devops/mcp …`). Never fall back to guessing work item content.

Git (branches, commits, push) is done with the local `git` CLI, using the user's credentials.

**Branch strategy:** three long-lived branches `development → staging → prod` (see the `devops-github-actions` skill). Agents branch from and open PRs to the **integration branch** (`development`) only. `staging` and `prod` change only through release PRs the PO merges.

## Scrum process mapping

| Level | Work item type | Who writes it |
|---|---|---|
| Goal | Epic / Feature | PO |
| What to build | **Product Backlog Item (PBI)**: Description + **Acceptance Criteria** | PO |
| How to build it | **Task** (child of the PBI): one use case in **one repository**, one pull request. Title `T<n> — API: …` or `T<n> — Web: …`, tag `repo-api` or `repo-web` | planner (agents) |

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
| `agent-ready` | **PO**, on a **PBI** | Build the whole PBI: every child Task (except `agent-hold`) is approved, one PR per Task in `T<n>` order. A PBI without Tasks is planned first and then built without a separate task review |
| `agent-ready` | **PO**, on a Task | Approve just this Task (when the PBI is not `agent-ready`) |
| `agent-hold` | PO, on a Task | Skip this Task even though its PBI is `agent-ready` |
| `agent-no-merge` | PO, on a PBI | Build all Tasks, but let me merge each PR myself (turns off the agent merge) |
| `agent-working` | agent | Claimed; a team run is in progress |
| `agent-pr-open` | agent | Pull request open (waiting for CI and verification, or for the PO) |
| `agent-merged` | agent, on a Task | Verified and merged into `development` by the agents (PBI-approved Tasks only) |
| `agent-done` | agent, on a PBI | All Tasks merged and the PBI's criteria verified. The PO checks the listed manual items and closes the PBI (replaces `agent-ready`) |
| `needs-human` | agent | Blocked: question, failed build, or failed review. See the latest comment |
| `repo-api` / `repo-web` | agent, on new Tasks | Which repository the task changes (backend / frontend). Set at planning, never removed |

Rules:
- Change tags by editing `System.Tags` as a whole: read the current value, add/remove tags, write it back. Keep tags the PO added.
- Never add `agent-ready` yourself, never close or delete work items, never change PBI state or fields other than tags.
- **Approval rule:** an agent may write code for a Task only if the Task is tagged `agent-ready`, **or** its parent PBI is tagged `agent-ready` and the Task is not tagged `agent-hold`. `agent-ready` on a PBI is the PO's approval of all its Tasks, including Tasks the agents create for it.
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

PBIs approved to build (`agent-ready` on the PBI):

```sql
SELECT [System.Id] FROM WorkItems
WHERE [System.TeamProject] = @project
  AND [System.WorkItemType] = 'Product Backlog Item'
  AND [System.Tags] CONTAINS 'agent-ready'
  AND [System.State] <> 'Done'
ORDER BY [Microsoft.VSTS.Common.BacklogPriority] ASC
```

Their candidate Tasks: read each PBI with relations (`wit_work_item get`, `expand: relations`) and take the child Tasks in state To Do that are not tagged `agent-hold`, `agent-working`, `agent-pr-open` or `needs-human`.

Tasks approved one by one:

```sql
SELECT [System.Id] FROM WorkItems
WHERE [System.TeamProject] = @project
  AND [System.WorkItemType] = 'Task'
  AND [System.Tags] CONTAINS 'agent-ready'
  AND [System.Tags] NOT CONTAINS 'agent-working'
  AND [System.State] = 'To Do'
ORDER BY [System.Id] ASC
```

Tasks ready to build = both sets together.

Pick order among ready tasks: finish a PBI before starting the next (the parent PBI's `BacklogPriority` first), then the `T<n>` number in the title. Skip a task whose predecessor tasks are not Done (or not at least `agent-pr-open` if the PO has said stacking is fine). A Web task therefore waits until the API task it depends on is merged.

## Git and pull request conventions

- Work in the task's repository (`repo-api` → backend, `repo-web` → frontend; local folder from the settings). Branch, commit, push and PR all happen there.
- Branch: `agent/<taskId>-<short-slug>` from an up-to-date integration branch (`git -C <repo> switch development && git -C <repo> pull`).
- Commits: imperative subject, then a line `AB#<taskId>` so Azure Boards links the commit.
- Never commit to or push `development`, `staging` or `prod`. Never force-push someone else's branch.
- PR title: `AB#<taskId> <task title>`. Target: `development`, never `staging` or `prod`. Description:

```markdown
## What
<2–4 bullets from the developer hand-off>

## How to test
<steps; API https://localhost:5001, Angular https://localhost:4200 (a Web task needs the backend repo running)>

## Tests
<tester report: added tests, result>

## Review
<reviewer verdict + remaining non-blocking findings>

Fixes AB#<taskId>
```

(`Fixes AB#` closes the task when the PR is merged into the default branch `development`: Azure Boards GitHub app on GitHub, connected to **both** repos; "Complete associated work items" on Azure Repos.)

- The task is linked through `workItems` on create (and `AB#<id>` in the description).
- The agents never approve, vote on, set auto-complete or bypass policies on pull requests (`vote`, `autoComplete`, `bypassPolicy` are off-limits).
- **Merging:** a PR is merged by the PO, except for Tasks approved through their PBI's `agent-ready` (and no `agent-no-merge`): `ado-deliver` then merges into `development` with a merge commit (no squash) after green CI and a PASS from the `verifier` agent. Never into `staging` or `prod`.
