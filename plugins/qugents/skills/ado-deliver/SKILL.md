---
name: ado-deliver
description: Builds one approved Azure DevOps Task end to end with the agent team - picks the next Task tagged agent-ready (or a given id), claims it, creates a branch, runs fullstack-developer, tester and reviewer, then commits, pushes, opens a linked pull request and updates the work item. Use when the user says take the next task, deliver task 456, work the backlog, or similar.
---

# Deliver an approved task

Load the `ado-backlog` skill first: it has the project settings, tools, tags, WIQL and git/PR conventions. This skill runs in the **main session**: it talks to Azure DevOps and git, and orchestrates the agents. The agents themselves never touch Azure DevOps.

Input: a Task id (`/qugents:ado-deliver 456`) or nothing (= the next ready task). One task per run. To do several, run it again after the user has seen the result (ideally after `/clear`).

## 1. Pick and check

- With an id: read it and verify it is a Task tagged `agent-ready` in state To Do. If not, stop and say why.
- Without an id: run the "Tasks ready to build" WIQL, apply the pick order and dependency rule from `ado-backlog`, take the first one. If none: tell the user and stop.
- Read the task (description = the spec), its parent PBI (title, description, acceptance criteria) and comments.
- The working tree must be clean (`git status`). If not, stop and ask the user.

## 2. Claim

- Tags: remove `agent-ready`, add `agent-working`. State: In Progress.
- Comment: `Agent started on this task (branch agent/<id>-<slug>).`
- `git switch <default> && git pull && git switch -c agent/<id>-<slug>`

## 3. Build with the team

1. **Developer.** Spawn `qugents:fullstack-developer` with the task title, the full task description, the parent PBI's acceptance criteria, and: "Implement exactly this task. Do not commit."
2. **Tester.** Spawn `qugents:tester` with the task spec, the developer's hand-off (including "For the tester") and: "Write and run the tests for this change. Do not commit."
3. **Reviewer.** Spawn `qugents:reviewer` with the task spec and: "Review the uncommitted changes against the default branch (git diff)."
4. **Fix loop (max 2 rounds).** If the tester reports bugs or the reviewer says *Request changes*: send the findings (Blockers and Majors only) to `qugents:fullstack-developer` to fix, then run tester and reviewer again. Structural findings may be checked with `qugents:architecture` first.
5. If it is still not green after 2 rounds, or the developer stops with a question: go to **Blocked**.

Before committing, run `dotnet build`, `dotnet test` and (if the frontend changed) `npx ng build` / `npx ng test --watch=false` yourself, and check that no secrets or `app.db` files are staged.

## 4. Pull request

- `git add` the changed files, commit (`<imperative summary>` + blank line + `AB#<id>`), `git push -u origin agent/<id>-<slug>`.
- `repo_pull_request_write create` with the title and description template from `ado-backlog`, `sourceRefName: refs/heads/agent/<id>-<slug>`, `targetRefName: refs/heads/<default>` and `workItems: "<id>"` (links the task). Not a draft.
- Tags: remove `agent-working`, add `agent-pr-open`. Leave the state In Progress (it closes when the PO completes the PR with "Complete associated work items").
- Comment: `PR !<prId> is ready for review: <url>. Tests: <passed>/<total>. Review: <verdict>.`

## Blocked

- Commit and push what exists on the branch only if it builds; otherwise leave the working tree for the user and say so.
- Tags: remove `agent-working`, add `needs-human`. Comment with: what was done, what failed (exact error or open question), and what the PO should decide.
- Do not open a PR for broken work.

## Finish

Tell the user in a few lines: task id and title, PR link (or why it is blocked), test result, review verdict, anything the PO must decide. Do not start the next task unasked.

## Rules

- Only `agent-ready` tasks are built. Never merge, approve or complete a PR. Never push to the default branch.
- Never change acceptance criteria to fit the implementation; if the spec is wrong, block and ask.
- Keep the PO's tags and comments; edit only the agent tags listed in `ado-backlog`.
