---
name: ado-plan
description: Plans Azure DevOps Product Backlog Items for the agent team - reads PBIs tagged agent-plan (or a given PBI id), has the planner agent break each into tasks, and creates them as child Tasks tagged agent-proposed for the product owner to approve. Use when the user says plan the backlog, plan PBI 123, break down this item, or similar.
---

# Plan backlog items into agent tasks

Load the `ado-backlog` skill first: it has the project settings, tools, fields, tags and WIQL. This skill runs in the **main session** (it talks to Azure DevOps); the planning itself is done by the `qugents:planner` agent.

Input: a PBI id (`/qugents:ado-plan 123`) or nothing, which means every PBI tagged `agent-plan`.

## Steps per PBI

1. **Read.** `wit_work_item get` with relations: title, description, acceptance criteria, tags, parent Feature/Epic (read its title/description for context), existing child Tasks, and `list_comments` (the PO may have answered earlier questions there).
2. **Check readiness.** If acceptance criteria are missing or the item is too vague to plan, do not invent them: add a comment with 1–5 concrete questions, add tag `needs-human`, and move to the next PBI.
3. **Plan.** Spawn `qugents:planner` with: the PBI title, description and acceptance criteria verbatim; the parent's goal; the PO's comments; the existing child tasks (plan only the gap); the backend and frontend repository folders from the settings; and the instruction "Output tasks in your standard format. Each task must be one pull request in one repository (API or Web)."
4. **Create tasks.** One `wit_work_item_write add_child` call under the PBI with all tasks in order (`workItemType: "Task"`, `format: "Markdown"`):
   - Title: `T<n> — API: <imperative title>` or `T<n> — Web: <imperative title>`
   - Description (Markdown): Repo, Why, Acceptance criteria (Given/When/Then list), Backend or Frontend, API contract, Test cases, Depends on. Keep the planner's content as it is.
   Then one `update_batch` on the new ids: `/fields/System.Tags` = `agent-proposed; repo-api` or `agent-proposed; repo-web` and `/fields/Microsoft.VSTS.Scheduling.RemainingWork` = 2 (S), 4 (M) or 8 (L). `add_child` cannot set these itself.
5. **Link dependencies.** For "Depends on: T<k>", link the new task to T<k> as successor/predecessor.
6. **Update the PBI.** Tags: remove `agent-plan`, add `agent-planned`. Comment:

```
Agent plan: <n> tasks created (T1–T<n>), tagged agent-proposed.
Next step (PO): review/edit the tasks and change agent-proposed → agent-ready on the ones to build.
Open questions: <planner's open questions, or "none">
Risks: <planner's risks, or "none">
```

   If the planner has open questions that block a task, still create the other tasks, and tag the PBI `needs-human` as well.

## Rules

- Never tag anything `agent-ready`. That is the PO's approval.
- Never edit the PBI's description, acceptance criteria or state.
- Re-running on a PBI that already has `agent-planned` only adds tasks for new scope the PO asked for in a comment; it never duplicates existing tasks.
- Finish with a short summary to the user: PBIs planned, tasks created (ids + titles), questions raised.
