# Loop Protocol

The harness operates as a continuous loop over a task list. The agent picks one task, completes it through all stages, commits, and picks the next. This continues until all tasks are done.

## Loop Stages

```
PICK → SPEC → PLAN → BUILD → VERIFY → REVIEW → COMMIT → LEARN → NEXT
  ↑                                                              │
  └──────────────────────────────────────────────────────────────┘
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
Return to PICK. The loop continues until `task next` reports all tasks complete.

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
