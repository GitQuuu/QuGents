---
name: ado-deliver
description: Builds one approved Azure DevOps Task end to end with the agent team - picks the next approved Task (tagged agent-ready, or a child of a PBI tagged agent-ready), or a given Task or PBI id, claims it, creates a branch, runs fullstack-developer, tester and reviewer, then commits, pushes, opens a linked pull request and updates the work item. Use when the user says take the next task, deliver task 456, work the backlog, or similar.
---

# Deliver an approved task

Load the `ado-backlog` skill first: it has the project settings, tools, tags, WIQL and git/PR conventions. This skill runs in the **main session**: it talks to Azure DevOps and git, and orchestrates the agents. The agents themselves never touch Azure DevOps.

Input: a Task id (`/qugents:ado-deliver 456`), a PBI id (`/qugents:ado-deliver 233` = the next Task of that PBI) or nothing (= the next ready task). One task per run, because the next Task builds on the merged result of the previous one: after the PO has merged the PR, run it again (ideally after `/clear`).

## 1. Pick and check

- With a Task id: read it and check the **approval rule** from `ado-backlog` (Task `agent-ready`, or parent PBI `agent-ready` and Task not `agent-hold`) and state To Do. If not, stop and say why.
- With a PBI id: the PBI must be tagged `agent-ready`. If it has no child Tasks yet, run the `ado-plan` steps for it first (the PBI's `agent-ready` approves the new Tasks). Then take its first Task by `T<n>` that is To Do, not `agent-hold`, and whose predecessors are Done. If the next Task waits for an open PR, say which PR the PO must merge and stop.
- Without an id: run the "Tasks ready to build" queries, apply the pick order and dependency rule from `ado-backlog`, take the first one. If none: tell the user and stop.
- Read the task (description = the spec), its parent PBI (title, description, acceptance criteria) and comments.
- **Pick the repository**: tag `repo-api` → backend repo, `repo-web` → frontend repo (fallback: the `API:`/`Web:` title prefix). Resolve its local folder from the settings; below, `<repo>` is that folder. If the tag is missing or the folder does not exist, stop and ask.
- The working tree of `<repo>` must be clean (`git -C <repo> status`). If not, stop and ask the user.

## 2. Claim

- Task tags: remove `agent-ready` / `agent-proposed`, add `agent-working`. State: In Progress. Leave the PBI's `agent-ready` in place until all its Tasks are done.
- Comment: `Agent started on this task (branch agent/<id>-<slug>).`
- `git -C <repo> switch development && git -C <repo> pull && git -C <repo> switch -c agent/<id>-<slug>` (the integration branch from `CLAUDE.md`)

## 3. Build with the team

1. **Developer.** Spawn `qugents:fullstack-developer` with the task title, the full task description, the parent PBI's acceptance criteria, and: "This is an API|Web task. Work only in <repo> (you may read the other repo). Implement exactly this task. Do not commit."
2. **Tester.** Spawn `qugents:tester` with the task spec, the developer's hand-off (including "For the tester") and: "Work only in <repo>. Write and run the tests for this change. Do not commit."
3. **Reviewer.** Spawn `qugents:reviewer` with the task spec and: "Review the uncommitted changes in <repo> against the integration branch (git -C <repo> diff)."
4. **Fix loop (max 2 rounds).** If the tester reports bugs or the reviewer says *Request changes*: send the findings (Blockers and Majors only) to `qugents:fullstack-developer` to fix, then run tester and reviewer again. Structural findings may be checked with `qugents:architecture` first.
5. If it is still not green after 2 rounds, or the developer stops with a question: go to **Blocked**.

Before committing, verify in `<repo>` yourself: API task `dotnet build` + `dotnet test`; Web task `npx ng build` + `npx ng test --watch=false`. Check that no secrets, `app.db` or `certs/` files are staged, and that the other repo has no changes.

## 4. Pull request

- In `<repo>`: `git add` the changed files, commit (`<imperative summary>` + blank line + `AB#<id>`), `git push -u origin agent/<id>-<slug>`.
- Open the PR to `development` with the title and description template from `ado-backlog`:
  - **GitHub:** `gh pr create --repo <owner/repo of the task's repository> --base development --head agent/<id>-<slug> --title "AB#<id> <title>" --body-file <file>`.
  - **Azure Repos:** `repo_pull_request_write create` with `sourceRefName: refs/heads/agent/<id>-<slug>`, `targetRefName: refs/heads/development`, `workItems: "<id>"`.
  Not a draft. Never target `staging` or `prod`.
- Tags: remove `agent-working`, add `agent-pr-open`. Leave the state In Progress (`Fixes AB#<id>` closes it when the PO merges the PR).
- Comment: `PR !<prId> is ready for review: <url>. Tests: <passed>/<total>. Review: <verdict>.`

## Blocked

- Commit and push what exists on the branch only if it builds; otherwise leave the working tree for the user and say so.
- Tags: remove `agent-working`, add `needs-human`. Comment with: what was done, what failed (exact error or open question), and what the PO should decide.
- Do not open a PR for broken work.

## Finish

Tell the user in a few lines: task id and title, PR link (or why it is blocked), test result, review verdict, anything the PO must decide. If the Task belongs to a PBI tagged `agent-ready`, also say which Task is next and that it starts when this PR is merged and `/qugents:ado-deliver` is run again. Do not start the next task unasked. When the last Task of the PBI has its PR open, comment on the PBI: `All <n> tasks have pull requests; the PBI is ready for your review.`

## Rules

- Only `agent-ready` tasks are built. Never merge, approve or complete a PR. Never push to `development`, `staging` or `prod`.
- Never change acceptance criteria to fit the implementation; if the spec is wrong, block and ask.
- Keep the PO's tags and comments; edit only the agent tags listed in `ado-backlog`.
