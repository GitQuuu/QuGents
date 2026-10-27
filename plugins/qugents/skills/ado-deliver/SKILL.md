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

**Context pack.** Agents start without context and lose time exploring the codebase. Build one context pack per Task and put it at the top of every agent prompt: the repository folder; the Task spec and acceptance criteria; the exact files and folders the Task names (Backend/Frontend section, API contract); one comparable existing slice (its folder path); and, once the developer is done, its list of changed files and `git -C <repo> diff --stat`. Tell each agent: "Start from these files; explore further only if they are not enough."

0. **Bug first.** If the Task fixes a bug (work item type Bug, or the title starts with "Fix"), spawn `qugents:tester` first: "Reproduce this bug with a failing test through the public API in <repo>; run it and show the failure. Do not change production code." Give the developer that test as the definition of done.
1. **Developer.** Spawn `qugents:fullstack-developer` with the task title, the full task description, the parent PBI's acceptance criteria, and: "This is an API|Web task. Work only in <repo> (you may read the other repo). Implement exactly this task. Do not commit."
2. **Tester.** Spawn `qugents:tester` with the task spec, the developer's hand-off (including "For the tester"), both repository folders and: "Work only in <repo>. Write and run the tests for this change (behaviour through the public API, real database; for a Web task also walk the user flow with Playwright MCP). Map every acceptance criterion to a passing test or mark it Manual. No mutation testing in a Task. Do not commit."
3. **Reviewer and security, in parallel.** They read the same diff independently, so spawn both in **one message** (two Agent calls) and wait for both:
   - `qugents:reviewer` with the task spec and: "Review the uncommitted changes in <repo> against the integration branch (git -C <repo> diff) for correctness, robustness, performance and test adequacy. Security is reviewed by qugents:security in parallel; do not duplicate it. Do not build or run tests."
   - `qugents:security` in review mode with the task spec, its acceptance criteria (including the security requirements from planning) and: "Review the uncommitted changes in <repo> for security. Do not run the test suite."
4. **Fix loop: developer and security spar (max 2 rounds).** If the tester reports bugs, the reviewer says *Request changes*, or security says FAIL: send the findings (reviewer Blockers and Majors, security Critical and High, each with its scenario and suggested fix) to `qugents:fullstack-developer` to fix. If the developer disagrees with a security finding or needs a different fix, send its answer back to `qugents:security` and let it confirm or adjust; relay between them until they agree. Then re-run **only what had findings**: the tester (filtered to the module) if tests or behaviour changed, the reviewer if it requested changes, security if it said FAIL (it re-checks its findings and what the fix touched); run those re-checks in parallel too. Structural findings may be checked with `qugents:architecture` first. Security Medium and Low findings do not loop; they go in the PR.
5. If it is still not green after 2 rounds, security still has an open Critical or High finding, or the developer stops with a question: go to **Blocked** (`needs-human`, with the open findings in the comment). If the team solved everything, continue: the Task is merged as usual without waiting for the PO (section 5).

Before committing, build and run **this Task's filtered tests** once yourself (API: `dotnet build` + the tester's filter; Web: `npx ng build` + the feature's specs). Do **not** run the full suite locally: CI runs it on the PR and is the gate. Run the full suite locally only when the Task changed shared code (`Common/`, base classes, DI extensions, `Program.cs`, middleware). Check that no secrets, `app.db` or `certs/` files are staged, and that the other repo has no changes.

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

1. **CI.** `gh pr checks <n> --repo <owner/repo> --watch --fail-fast` (allow up to 30 minutes). A failing check → one fix round (developer fixes, tester re-runs, push), then wait again. Still red → **Blocked**. If the repo has no checks at all, run the full suite locally once instead and say so in the PR comment.
2. **Acceptance check (you, no agent).** Use the tester's mapping: every acceptance criterion of the Task (including the security requirements) has a named test that passed in CI, or is marked Manual with what the PO should look at. A criterion without evidence → one round where the tester adds the test (push, CI again). Still missing → **Blocked**. The verifier agent is not run per Task; it checks the whole PBI at the end.
3. **Merge with a merge commit** (never squash or rebase, so every Task stays visible as its own branch in the history): `gh pr merge <n> --repo <owner/repo> --merge --delete-branch --subject "Merge AB#<id> <title>" --body "Fixes AB#<id>"`. Never `--admin`, never bypass a failing or missing required check. If the repo does not allow merge commits, stop and ask the PO to enable *Allow merge commits* (Settings → General → Pull Requests). Merge conflict: merge `origin/development` into the branch, build and test, push, and go back to step 1 (once); otherwise **Blocked**.
4. **After the merge.** `git -C <repo> switch development && git -C <repo> pull`, then `git -C <repo> branch -d agent/<id>-<slug>` and `git -C <repo> fetch --prune` (`--delete-branch` already removed the remote branch). Task tags: remove `agent-pr-open`, add `agent-merged`. If the Task is not Done a minute later (Azure Boards app not connected), set its state to Done. Comment: `Merged PR #<n>: CI green, <n> criteria covered by tests, <m> manual.`, listing any Manual checks.
5. **Next Task.** Pick the next Task of the same PBI (pick order and dependencies from `ado-backlog`) and go back to section 1 without asking. A Web Task whose API Task was just merged can start now.

### PBI done

When every Task of the PBI (except `agent-hold`) is Done:

- **Mutation testing, once per PBI.** Spawn `qugents:tester`: "Run mutation testing on everything this PBI changed in <repo> (`dotnet stryker --since:<the development commit before the PBI's first merge>`; StrykerJS for the frontend if set up). Add tests for surviving mutants that represent real behaviour, on branch `agent/<pbiId>-mutation-tests`." If it added tests: commit (`AB#<pbiId>`), PR, CI, merge as in section 5 (no security review needed for test-only changes).
- Spawn `qugents:verifier` for the **PBI's** acceptance criteria, across both repos on `development`, with the Tasks' criteria-to-test mappings and the merged PRs.
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
- Agents never approve or vote on PRs, never use `--admin` or bypass checks, never merge into `staging` or `prod`, and never push directly to `development`, `staging` or `prod`. The only merge an agent makes is section 5: a PBI-approved Task (or the PBI's mutation-test PR), into `development`, with green CI, every acceptance criterion covered by a passing test or marked Manual, and a security PASS.
- Never change acceptance criteria to fit the implementation; if the spec is wrong, block and ask.
- Keep the PO's tags and comments; edit only the agent tags listed in `ado-backlog`.
