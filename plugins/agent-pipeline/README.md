# agent-pipeline

An autonomous task pipeline. It takes the project's tasks (a file in the repo or GitHub issues), runs each item through implementation, review and fixing until tests are green, committing as it goes.

## How it works

```
orchestrator
  ├── planner          (epic too big → ordered sub-tasks)
  ├── implementer      (writes the code + tests)
  ├── doc-sweeper      (brings the docs in step with the change, before review)
  ├── code-reviewer    (strict PASS / FAIL verdict)
  └── fixer            (addresses reported issues, max 3 iterations)

triager                (item not ready yet → ready, or a question for a human)
```

The orchestrator never writes code. It sizes each item just in time, spawns subagents, runs the project's tests itself as the source of truth, and keeps the task source updated so any fresh session can resume the run.

## Agents

| Agent | Model | Role |
|-------|-------|------|
| `orchestrator` | opus | Coordinates the run, tracks state, commits, reports |
| `planner` | fable | Read-only. Breaks an epic into ordered tasks with a definition of done |
| `implementer` | sonnet (high effort) | Implements exactly one task, with tests |
| `doc-sweeper` | sonnet (high effort) | Fresh eyes on the change: greps every subject it touches across the docs and READMEs, updates stale statements, checks each claim against the code. Edits docs only |
| `code-reviewer` | fable | Reviews the diff, may run any check, changes nothing; returns `VERDICT: PASS` or `VERDICT: FAIL` |
| `fixer` | opus | Fixes only the reported issues, no rewrites |
| `triager` | opus | Reviews one item not ready yet; makes it ready, or parks it with a question for a human. Changes only the task source, never the repository |

## Usage

```
Use the agent-pipeline:orchestrator agent
```

Installed as a plugin, the agents are namespaced under the plugin name, so they are `agent-pipeline:orchestrator`, `agent-pipeline:planner`, `agent-pipeline:implementer`, `agent-pipeline:doc-sweeper`, `agent-pipeline:code-reviewer` and `agent-pipeline:fixer`. The orchestrator spawns the others by their namespaced name and falls back to the short name if you installed them as plain project agents (copied into `.claude/agents/`) instead.

## Unattended runs

`bin/run-pipeline` runs the pipeline without supervision: it starts one headless orchestrator session per item (an epic with its sub-tasks, or a standalone ticket such as a bug), so each item begins with a fresh context, and loops until nothing is selectable. Then, if the task source has a Triage section, it starts one headless triager session per item not ready yet: an item made ready goes back to the orchestrator, an item that needs a human decision is parked with its question. It stops when nothing is selectable and nothing is left to triage. A crashed or interrupted session is simply followed by a new one, which resumes from the task source.

```
<plugin dir>/bin/run-pipeline -C ~/code/my-project
```

Each session runs `claude -p --agent agent-pipeline:orchestrator --permission-mode auto` in single-item mode (`--agent agent-pipeline:triager` for a triage); the agent ends its report with `PIPELINE_STOP: <reason>`, which the runner reads:

| Session | Reason | Runner |
|---------|--------|--------|
| orchestrator | `item-done` | starts the next orchestrator session |
| orchestrator | `nothing-selectable` | starts a triager session (exits 0 with `--no-triage`) |
| triager | `item-ready` | starts an orchestrator session |
| triager | `item-needs-decision` | starts the next triager session |
| triager | `nothing-selectable` | exits 0 |
| either | `error` | exits 1, a human must look |
| either | no stop line (crash) | retries after a pause, gives up after 3 in a row (exit 2) |

A parked item is triaged again only once a human has answered on it, so the runner never loops on it. To pick answers up, run the runner again, or schedule it.

Options: `--max-runs`, `--max-retries`, `--permission-mode`, `--prompt` (extra instruction for every orchestrator session), `--no-triage`, `--log-dir` (default `~/.local/state/agent-pipeline/<project>`), and claude arguments after `--`. Headless claude kills background subagents 10 minutes after the orchestrator ends its turn, so the runner raises that wait to 2 hours (`CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`, overridable from the environment). Run it in `tmux` or as a service on the machine that does the work.

## Task source

Each project says where its tasks live, in a dedicated file referenced from its `CLAUDE.md`, for example:

```
Agent pipeline task source: .claude/pipeline.md
```

The wording is free: the orchestrator looks for any clear reference to the file, and stops and reports if it finds several candidates or an ambiguous one.

The file is free-form. It tells the orchestrator how to pick the next task, how to mark it started / done / blocked, how to deliver (branch, push, pull request), where to record an epic's breakdown, where to leave notes, where the polish item gathering an epic's non-blocking review notes goes, optionally how items not ready yet are triaged, and optionally how many items to process per run. Items are epics and standalone tickets (bugs, tasks not attached to an epic) alike. By default a run goes on until no selectable item is left, re-reading the task source before each pick so items made ready during the run are taken too; it stops earlier only on an explicit restriction in the prompt ("only epic #42"), or on the task source's Scope. Work handed over in a pull request but not merged yet is *delivered*: skipped by later runs, reported with its link. Whatever it leaves out falls back to the defaults below. The task source is the persistent state of the run: any fresh session resumes from it.

### Triage

Optional. Without a Triage section, the runner stops once nothing is selectable and items not ready yet wait for a human. With one, the `triager` reviews them one at a time, against the criteria the section points to (typically a review guide: what to check per item type, which findings need a human). It settles what has an obvious answer, writes it into the item, and makes it ready; a decision that is costly to reverse, changes the scope of work already started, or has no obvious answer is asked on the item instead, which is parked until a human answers. The answer is written into the item on the next triage. Defaults for GitHub issues: every triager comment starts with `<!-- agent-pipeline:triage -->`, the only way to tell it from a human's when the pipeline posts with your account; a parked issue has an answer when a comment without the marker follows the last marked one; the triager never assigns the issue and changes its status last, so an interrupted triage is simply started again.

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
- Follow-ups: a standalone `Polish: follow-ups from #<epic>` issue, created only for an epic broken down into tasks.
- Triage: issues labelled `backlog`, oldest first, reviewed against docs/issue-review.md. Ready: swap `backlog` for `todo`. Needs a human decision: swap it for `needs-decision`; my answer is a comment after your question.
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
- At the end, the orchestrator reports a status table and ends with a `PIPELINE_STOP: <reason>` line.
