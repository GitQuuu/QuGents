---
name: ado-deliver
description: Builds approved Azure DevOps Tasks end to end with the agent team - picks the next approved Task (tagged agent-ready, or a child of a PBI tagged agent-ready) or a given Task or PBI id, claims it, branches, runs fullstack-developer, tester, reviewer and security (developer and security spar until security findings are fixed), opens a linked pull request, and for PBI-approved tasks also waits for CI, has the verifier check the acceptance criteria, merges into development and continues with the next task until the PBI is done. Use when the user says build PBI 233, take the next task, deliver task 456, work the backlog, or similar.
---

# Deliver an approved task

Load the `ado-backlog` skill first: it has the project settings, tools, tags, WIQL and git/PR conventions. This skill runs in the **main session**: it talks to Azure DevOps and git, and orchestrates the agents. The agents themselves never touch Azure DevOps.

Input: a PBI id (`/qugents:ado-deliver 233`), a Task id (`/qugents:ado-deliver 456`) or nothing (= the next ready task).

- **PBI approved (`agent-ready` on the PBI):** the run builds, verifies and merges the Tasks one after another (sections 1–5) until the PBI is done, a Task is blocked, or the user stops it. The PO approved the whole PBI, so no PR waits for a human. A PBI tagged `agent-no-merge` is built the same way but every PR waits for the PO.
- **Task approved on its own (`agent-ready` on the Task):** one Task per run; the PR waits for the PO (sections 1–4).

If the PBI already has a Task tagged `agent-pr-open` (from an earlier run), start with section 5 for that PR.

Between Tasks, keep only a short summary of the previous one; the details live on the work item and the PR.

## 1. Pick and check

- With a Task id: read it and check the **approval rule** from `ado-backlog` (Task `agent-ready`, or parent PBI `agent-ready` and Task not `agent-hold`) and state To Do. If not, stop and say why.
- With a PBI id: the PBI must be tagged `agent-ready`. If it has no child Tasks yet, run the `ado-plan` steps for it first (the PBI's `agent-ready` approves the new Tasks). Then take its first Task by `T<n>` that is To Do, not `agent-hold`, and whose predecessors are Done. If the next Task waits for an open PR of this PBI: go to section 5 for that PR (or, with `agent-no-merge`, say which PR the PO must merge and stop).
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
4. **Security.** Spawn `qugents:security` in review mode with the task spec, its acceptance criteria (including the security requirements from planning) and: "Review the uncommitted changes in <repo> for security."
5. **Fix loop: developer and security spar (max 2 rounds).** If the tester reports bugs, the reviewer says *Request changes*, or security says FAIL: send the findings (reviewer Blockers and Majors, security Critical and High, each with its scenario and suggested fix) to `qugents:fullstack-developer` to fix. If the developer disagrees with a security finding or needs a different fix, send its answer back to `qugents:security` and let it confirm or adjust; relay between them until they agree. Then run tester, reviewer and security again (security re-checks its findings and what the fix touched). Structural findings may be checked with `qugents:architecture` first. Security Medium and Low findings do not loop; they go in the PR.
6. If it is still not green after 2 rounds, security still has an open Critical or High finding, or the developer stops with a question: go to **Blocked** (`needs-human`, with the open findings in the comment). If the team solved everything, continue: the Task is merged as usual without waiting for the PO (section 5).

Before committing, verify in `<repo>` yourself: API task `dotnet build` + `dotnet test`; Web task `npx ng build` + `npx ng test --watch=false`. Check that no secrets, `app.db` or `certs/` files are staged, and that the other repo has no changes.

## 4. Pull request

- In `<repo>`: `git add` the changed files, commit (`<imperative summary>` + blank line + `AB#<id>`), `git push -u origin agent/<id>-<slug>`.
- Open the PR to `development` with the title and description template from `ado-backlog`:
  - **GitHub:** `gh pr create --repo <owner/repo of the task's repository> --base development --head agent/<id>-<slug> --title "AB#<id> <title>" --body-file <file>`.
  - **Azure Repos:** `repo_pull_request_write create` with `sourceRefName: refs/heads/agent/<id>-<slug>`, `targetRefName: refs/heads/development`, `workItems: "<id>"`.
  Not a draft. Never target `staging` or `prod`.
- Tags: remove `agent-working`, add `agent-pr-open`. Leave the state In Progress.
- Comment: `PR #<n>: <url>. Tests: <passed>/<total>. Review: <verdict>.`
- Task approved on its own, or PBI tagged `agent-no-merge`: stop here and go to **Finish**. The PO merges.

## 5. Verify and merge (PBI-approved Tasks only)

Only when **all** hold: the Task is approved through its PBI's `agent-ready`, the PBI is not tagged `agent-no-merge`, the PR's base is the integration branch (`development`), and the last security review was PASS (no open Critical or High finding).

1. **CI.** `gh pr checks <n> --repo <owner/repo> --watch --fail-fast` (allow up to 30 minutes). A failing check → one fix round (developer fixes, tester re-runs, push), then wait again. Still red → **Blocked**. If the repo has no checks at all, say so in the PR comment and rely on step 2.
2. **Verifier.** Spawn `qugents:verifier` with the task title and description, the Task's and the PBI's acceptance criteria, `<repo>` and the PR number. FAIL → one fix round (findings to `qugents:fullstack-developer`, tester, push, CI, verifier again). Still FAIL → **Blocked**.
3. **Merge with a merge commit** (never squash or rebase, so every Task stays visible as its own branch in the history): `gh pr merge <n> --repo <owner/repo> --merge --delete-branch --subject "Merge AB#<id> <title>" --body "Fixes AB#<id>"`. Never `--admin`, never bypass a failing or missing required check. If the repo does not allow merge commits, stop and ask the PO to enable *Allow merge commits* (Settings → General → Pull Requests). Merge conflict: merge `origin/development` into the branch, build and test, push, and go back to step 1 (once); otherwise **Blocked**.
4. **After the merge.** `git -C <repo> switch development && git -C <repo> pull`, then `git -C <repo> branch -d agent/<id>-<slug>` and `git -C <repo> fetch --prune` (`--delete-branch` already removed the remote branch). Task tags: remove `agent-pr-open`, add `agent-merged`. If the Task is not Done a minute later (Azure Boards app not connected), set its state to Done. Comment: `Verified (<n> criteria, <m> manual) and merged: PR #<n>.`, listing any Manual checks.
5. **Next Task.** Pick the next Task of the same PBI (pick order and dependencies from `ado-backlog`) and go back to section 1 without asking. A Web Task whose API Task was just merged can start now.

### PBI done

When every Task of the PBI (except `agent-hold`) is Done:

- Spawn `qugents:verifier` once more for the **PBI's** acceptance criteria, across both repos on `development`.
- PBI tags: remove `agent-ready`, add `agent-done`. Leave the PBI's state to the PO.
- Comment on the PBI: the verification table, the merged PRs, and the Manual checks the PO should do before closing the PBI.

## Blocked

- Commit and push what exists on the branch only if it builds; otherwise leave the working tree for the user and say so.
- Tags: remove `agent-working` (or `agent-pr-open`), add `needs-human`. Comment on the Task with: what was done, what failed (exact error, failing check or criterion) and what the PO should decide. Add a one-line comment on the PBI that the run stopped at this Task.
- Do not open a PR for broken work and never merge a PR that is not green and verified. Stop the run; do not skip ahead to later Tasks.

## Finish

Tell the user in a few lines: the Tasks handled with PR links and whether they were merged, test results, verification verdicts, Manual checks, and anything the PO must decide. For a Task-approved run or an `agent-no-merge` PBI, say that the next Task starts when the PR is merged and `/qugents:ado-deliver` is run again.

## Rules

- Only approved Tasks are built (approval rule in `ado-backlog`).
- Agents never approve or vote on PRs, never use `--admin` or bypass checks, never merge into `staging` or `prod`, and never push directly to `development`, `staging` or `prod`. The only merge an agent makes is section 5: a PBI-approved Task, into `development`, with green CI and a PASS from the verifier.
- Never change acceptance criteria to fit the implementation; if the spec is wrong, block and ask.
- Keep the PO's tags and comments; edit only the agent tags listed in `ado-backlog`.
