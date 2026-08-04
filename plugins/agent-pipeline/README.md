# agent-pipeline

An autonomous task pipeline. You hand it a task list, it runs each item through implementation, review and fixing until tests are green, committing as it goes.

## How it works

```
orchestrator
  ├── planner          (epic too big → ordered sub-tasks)
  ├── implementer      (writes the code + tests)
  ├── code-reviewer    (strict PASS / FAIL verdict)
  └── fixer            (addresses reported issues, max 3 iterations)
```

The orchestrator never writes code. It triages each item just in time, spawns subagents, runs the project's tests itself as the source of truth, and keeps the task file updated so any fresh session can resume the run.

## Agents

| Agent | Model | Role |
|-------|-------|------|
| `orchestrator` | opus | Coordinates the run, tracks state, commits, reports |
| `planner` | fable | Read-only. Breaks an epic into ordered tasks with a definition of done |
| `implementer` | sonnet | Implements exactly one task, with tests |
| `code-reviewer` | fable | Read-only. Reviews the diff, returns `VERDICT: PASS` or `VERDICT: FAIL` |
| `fixer` | sonnet | Fixes only the reported issues, no rewrites |

## Usage

Give the orchestrator a task list, either inline or in a file:

```
Use the agent-pipeline:orchestrator agent on tasks.md
```

Installed as a plugin, the agents are namespaced under the plugin name, so they are `agent-pipeline:orchestrator`, `agent-pipeline:planner`, `agent-pipeline:implementer`, `agent-pipeline:code-reviewer` and `agent-pipeline:fixer`. The orchestrator spawns the others by their namespaced name and falls back to the short name if you installed them as plain project agents (copied into `.claude/agents/`) instead.

The task file is the persistent state of the run:

- `- [ ]` pending, `- [x]` done
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
