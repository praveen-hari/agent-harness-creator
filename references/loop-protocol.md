# Loop Protocol

The harness operates as a continuous loop over a task list using an **orchestrator/sub-agent** model. The orchestrator manages task state, specs, plans, and reviews. Sub-agents handle the BUILD stage with fresh context windows — enabling the loop to run across 50+ tasks without context exhaustion.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  ORCHESTRATOR (main agent)                          │
│  • Reads task files (~5KB per iteration)            │
│  • Runs task.py commands                            │
│  • Writes specs, plans, reviews                     │
│  • Delegates BUILD to sub-agents                    │
│  • Decides when project is complete                 │
│                                                     │
│  PICK → SPEC → PLAN ─┐                             │
│    ↑                  │                             │
│    │    ┌─────────────▼──────────────┐              │
│    │    │  SUB-AGENT (fresh context) │              │
│    │    │  • Full context window     │              │
│    │    │  • Implements ## Plan      │              │
│    │    │  • Runs verify command     │              │
│    │    │  • Returns DONE or BLOCKED │              │
│    │    └─────────────┬──────────────┘              │
│    │                  │                             │
│    │    REVIEW ← VERIFY ←┘                          │
│    │      │                                         │
│    │    COMMIT → LEARN → NEXT                       │
│    │                       │                        │
│    └───────────────────────┘                        │
└─────────────────────────────────────────────────────┘
```

### Why Sub-Agents?

- **No context exhaustion**: Each task gets a fresh context window
- **Orchestrator stays lean**: ~5KB per loop iteration (task file + script output + project context)
- **Parallel-ready**: Multiple sub-agents could run concurrently (future)
- **Failure isolation**: A sub-agent crash doesn't lose orchestrator state

### Project Context

Every sub-agent receives `.codestudio/project-context.md` in its prompt. This file contains:

- **Stack**: Language, framework, key libraries and versions
- **Architecture**: Directory layout and layer relationships (e.g., routes → services → repositories)
- **Conventions**: Naming patterns, error handling style, testing approach
- **Boundaries**: Files/directories that should not be modified

Without this, sub-agents would guess at where to put code, what patterns to follow, and what not to touch. With it, every sub-agent produces code that fits the project.

**The orchestrator updates project-context.md during LEARN** when architectural decisions change (new patterns adopted, new boundaries established, stack changes).

## Loop Stages

```
PICK → SPEC → PLAN → BUILD(sub-agent) → VERIFY → REVIEW → COMMIT → LEARN → NEXT
  ↑                                                                          │
  └──────────────────────────────────────────────────────────────────────────┘
```

### PICK
Get the next eligible task from the index. The task manager enforces:
- Only one active task at a time
- Dependency checking (a task with `needs: ["T-001"]` won't be picked until T-001 is done)
- Status ordering: `todo` tasks are picked before `backlog`

### SPEC
Define what to build. For complex tasks, write a structured specification with acceptance criteria. For trivial tasks, a single sentence suffices. The agent writes this in `## What` of the task file.

### PLAN
Break the work into subtask checkboxes in `## Plan`. Each subtask should be completable in a single commit. Order by dependency. Skip for trivial tasks.

### BUILD
Implement one subtask at a time. Check the box when done. Write tests alongside or before code (TDD). This is where the bulk of agent work happens.

### VERIFY
Run the project's verification command (tests, lint, type-check). All checks must pass. On failure: fix, retry, max 3 attempts, then BLOCK the task.

### REVIEW
Self-review or delegate to the reviewer agent. Check correctness, security, maintainability. Write findings in `## Review`. If issues found: fix and re-review.

### COMMIT
Atomic commit with conventional message format. One subtask = one commit. Keep the git history clean and bisectable.

### LEARN
Record what happened: decisions, problems, solutions. Write to `## Log` in the task file and append to `progress.md`. This creates memory across sessions.

### NEXT
Return to PICK. The loop continues autonomously.

## Autonomous Completion

The orchestrator decides when the project is done. It does not stop at an arbitrary point.

### Auto-Promote Backlog

When all `todo` tasks are done but `backlog` items remain:
1. Evaluate each backlog item against the project goals
2. If needed to meet the original spec → promote to `todo`
3. If nice-to-have and spec is already met → skip
4. Continue the loop with promoted tasks

### Task Discovery

During BUILD and REVIEW, new work may be discovered:
- Missing error handling → `task add "Add error handling for X" --needs T-current`
- Untested edge case → `task add "Test edge case Y"`
- Dependency gap → `task add "Install/configure Z" --needs T-current`

The orchestrator adds these tasks and the loop picks them up naturally.

### Completion Criteria

The project is complete when ALL of:
1. All `todo` tasks are `done`
2. All `backlog` items evaluated (promoted or deliberately skipped)
3. Verify command passes on the full project
4. The original goals (from first task spec or progress.md) are satisfied

The orchestrator writes a final entry in progress.md:
```
PROJECT COMPLETE — [summary of what was built, key decisions, final state]
```

## State Machine

Tasks flow through these statuses:

```
backlog → todo → active → review → done
                   │                  │
                   ↓                  ↓
                 blocked           archive
                   │
                   ↓
                  todo (unblock)
```

- **backlog**: Ideas, future work. Not eligible for PICK.
- **todo**: Ready to work. Will be picked when dependencies are met.
- **active**: Currently being worked on. Only one at a time.
- **review**: Submitted for review. Can be approved (→done) or rejected (→active).
- **blocked**: Can't proceed. Needs external action or info.
- **done**: Complete. Can be archived to clean up the index.

## Task Files

Each task gets one markdown file: `.codestudio/tasks/<ID>.md`

Sections:
- `## What` — Specification. What to build, acceptance criteria.
- `## Plan` — Subtask checkboxes. Each is one commit's worth of work.
- `## Review` — Findings from code review. Issues and resolutions.
- `## Security` — Security notes (only if relevant).
- `## Log` — Decisions, problems encountered, solutions applied.

Cross-references between tasks use: "See T-001".

## Key Constraints

1. **Agent never edits index.json.** All state changes go through `task.py`.
2. **One active task.** Finish, block, or review before picking another.
3. **Commit per subtask.** Keeps history atomic and bisectable.
4. **Max 3 verify retries.** Block and move on if stuck.
5. **Progress.md is memory.** Write what future sessions need to know.
6. **BUILD is delegated.** Orchestrator never builds directly — use `runSubagent`.
7. **Loop until complete.** Don't stop at "all todo done" — evaluate backlog and project goals.
8. **Sub-agents don't run task.py.** Only the orchestrator manages task state.
