# agent-pipeline

An autonomous task pipeline. It takes the project's tasks (a file in the repo or GitHub issues), runs each item through implementation, review and fixing until tests are green, committing as it goes.

## How it works

```
orchestrator
  ├── planner          (epic too big → ordered sub-tasks)
  ├── implementer      (writes the code + tests)
  ├── code-reviewer    (strict PASS / FAIL verdict)
  └── fixer            (addresses reported issues, max 3 iterations)
```

The orchestrator never writes code. It triages each item just in time, spawns subagents, runs the project's tests itself as the source of truth, and keeps the task source updated so any fresh session can resume the run.

## Agents

| Agent | Model | Role |
|-------|-------|------|
| `orchestrator` | opus | Coordinates the run, tracks state, commits, reports |
| `planner` | fable | Read-only. Breaks an epic into ordered tasks with a definition of done |
| `implementer` | sonnet (high effort) | Implements exactly one task, with tests |
| `code-reviewer` | fable | Read-only. Reviews the diff, returns `VERDICT: PASS` or `VERDICT: FAIL` |
| `fixer` | opus | Fixes only the reported issues, no rewrites |

## Usage

```
Use the agent-pipeline:orchestrator agent
```

Installed as a plugin, the agents are namespaced under the plugin name, so they are `agent-pipeline:orchestrator`, `agent-pipeline:planner`, `agent-pipeline:implementer`, `agent-pipeline:code-reviewer` and `agent-pipeline:fixer`. The orchestrator spawns the others by their namespaced name and falls back to the short name if you installed them as plain project agents (copied into `.claude/agents/`) instead.

## Task source

Each project says where its tasks live, in a dedicated file referenced from its `CLAUDE.md`, for example:

```
Agent pipeline task source: .claude/pipeline.md
```

The wording is free: the orchestrator looks for any clear reference to the file, and stops and reports if it finds several candidates or an ambiguous one.

The file is free-form. It tells the orchestrator how to pick the next task, how to mark it started / done / blocked, how to deliver (branch, push, pull request), where to record an epic's breakdown, where to leave notes, and optionally how many items to process per run. Work handed over in a pull request but not merged yet is *delivered*: skipped by later runs, reported with its link. Whatever it leaves out falls back to the defaults below. The task source is the persistent state of the run: any fresh session resumes from it.

Without a reference in `CLAUDE.md`, give the orchestrator a task list inline or as a file (`Use the agent-pipeline:orchestrator agent on tasks.md`).

### Example: GitHub issues

```markdown
# Agent pipeline task source

Tasks are GitHub issues in acme/shop.

- Select: the oldest open issue labelled `todo` with no assignee. Process one issue per run.
- Start: assign it to me and swap the `todo` label for `in-progress`.
- Breakdown: native sub-issues of the ticket, labelled `in-progress`. Work through them in sub-issue order.
- Deliver: one branch per ticket, `ticket/<n>`. Close each sub-issue once its commit is pushed. When the whole ticket is green, push the branch and open a pull request with `Closes #<n>`; the ticket stays open until I merge.
- Blocked: comment the reason, add the `blocked` label, leave it open.
- Notes: comment on the affected issue.
```

Defaults: commit on the current branch without pushing or opening a pull request, assign to `@me` on start, native sub-issues for breakdowns, commit subjects ending in `(#<n>)`, close with the commit hash on done, comment + `blocked` label on failure. The orchestrator never pushes unless Deliver says to.

### Example: a file in the repo

```markdown
# Agent pipeline task source

Tasks are in docs/roadmap.md, processed top to bottom.
```

Defaults for a file:

- `- [ ]` pending, `- [x]` done (marked in the task's own commit)
- `- [ ] BLOCKED: <reason>` when a task failed 3 fix iterations
- Sub-tasks from the planner are written back under their epic before implementation starts
- `NOTE:` annotations flag remaining items affected by completed work

## Project rules

If the project defines rules (`CLAUDE.md`, `.claude/rules/`), they take precedence over the agents' defaults. The orchestrator passes the relevant ones into every subagent prompt, since subagents have no access to its conversation.

## Notes

- One task at a time. Tasks touching the same files are never parallelized.
- Tests are the source of truth: a reviewer PASS with failing tests is a FAIL.
- Fix loops are internal. A task that needed 3 iterations still gets one commit describing the shipped change.
- At the end, the orchestrator reports a status table and, if more than half the tasks needed fixes, an analysis of the recurring review failures with proposed prompt changes.
