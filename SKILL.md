---
name: harness-creator
description: 'Set up a .codestudio/ agent harness for any project. Use when: "set up harness", "bootstrap project", "initialize tasks", "create task management", onboarding a codebase, or starting a new project. Scans repos to auto-detect stack, generates loop protocol, seeds tasks. Works on empty repos (interview mode), existing repos (auto-detection), and re-runs safely (upgrade mode).'
argument-hint: 'Optional: path to the project root to scaffold'
---

# Harness Creator

Generate a `.codestudio/` harness that turns any project into an agent-friendly workspace. The harness provides a loop protocol: pick a task, spec it, plan it, build it, verify, review, commit, learn, repeat.

## When to Use

- Starting work on a new project
- Onboarding to an existing codebase
- Setting up task management for agent sessions
- The user says "set up harness", "create harness", "bootstrap project", or "initialize project tasks"
- Re-running to upgrade an existing harness

## Step 1 — SCAN

Scan the repository to detect project characteristics. Read the reference document for detection rules:

```
read_file: ./references/detection-rules.md
```

Then inspect the project root:

1. **List root directory** to find build files, configs, source directories
2. **Detect language/framework** from manifest files (package.json, requirements.txt, Cargo.toml, etc.)
3. **Detect test/lint/build tools** for verification steps
4. **Count source files** to gauge project complexity
5. **Check for .git** to enable/disable COMMIT stage
6. **Scan for security signals** (auth, JWT, crypto patterns)
7. **Check for UI files** (tsx, jsx, vue, svelte, components/)
8. **Check for existing `.codestudio/`** → upgrade mode

9. **Extract project context** for existing repos:
   - Read 2-3 representative source files → detect naming conventions, patterns, error handling
   - Read directory structure → infer architecture layers (routes → services → models, etc.)
   - Read config files → identify linting rules, formatting, CI expectations
   - Read `.gitignore`, lock files → identify boundaries (don't-touch files)

Record findings:
- `PROJECT_NAME`: directory name or package name
- `TIER`: L0/L1/L2/L3 based on test reality (see detection-rules.md for inference)
- `GATES`: ordered list of evidence gates (command + threshold + ratchet) per tier
- `HAS_GIT`: true/false (required — no git means no rollback)
- `SECURITY_FLAGS`: list of detected patterns
- `UI_PROJECT`: true/false
- `SOURCE_COUNT`: number of source files
- `UPGRADE_MODE`: true if `.codestudio/` exists
- `STACK`: language, framework, key libraries
- `ARCHITECTURE`: directory layout and layer relationships
- `CONVENTIONS`: naming, patterns, error handling style
- `BOUNDARIES`: files/dirs that should not be modified
- `COVERAGE_TOOL`: detected coverage tool (coverlet, c8, coverage.py, etc.)

## Step 2 — INTERVIEW

**If empty repo** (no source files detected):

Read the `interview-me` skill if available:
```
read_file: ~/.agents/skills/interview-me/SKILL.md
```

Ask the user one question at a time to determine:
1. What are you building? (product description)
2. Who is it for? (users, developers, internal)
3. What tech stack? (this sets VERIFY_STEPS, STACK)
4. What architecture? (monolith, microservices, serverless — sets ARCHITECTURE)
5. Any conventions you want enforced? (naming, patterns, linting)
6. What's the first feature to build?

Use answers to populate PROJECT_NAME, VERIFY_STEPS, STACK, ARCHITECTURE, CONVENTIONS, and generate initial tasks.

**If existing repo**: Confirm detected settings with the user:
- "I detected [Node.js + Jest + ESLint + TypeScript]. Verification steps: 1) lint, 2) typecheck, 3) test. Correct? Any steps to add/remove?"
- Ask for PROJECT_NAME if not obvious from package.json

**If upgrade mode**: Report what will be updated:
- "Existing harness found. I'll update task.py and instructions. Your tasks and progress will be preserved."

## Step 2.5 — PROPOSE (never skip)

Before generating files, present the user with a proposal. **Nothing is written until approved.**

Show:
1. **Detection summary** — what was found and the evidence for each conclusion (file paths, dependency names)
2. **Tier selection** — which tier (L0–L3) based on what actually exists:
   - **L0 Unverified**: no tests → build + lint only. Output labeled `UNVERIFIED`.
   - **L1 Diff-verified**: tests exist → tests + diff coverage on changed lines.
   - **L2 Verified**: good test suite → full suite + CVE scanning.
   - **L3 Hardened**: mature project → mutation testing + perf/leak budgets.
3. **Gate table** — each gate with: tool, command, threshold, status (ready / needs install)
4. **What is NOT proposed** — with a one-line rationale and cost for each omission. *This is the teaching surface — it's how users learn that diff coverage and mutation testing exist and what they cost.*
5. **Gaps** — missing tools, missing configs, and what the harness will do about each
6. **File list** — what will be created, what will be preserved

Ask: `Proceed? [y / adjust tier / adjust gates / explain <gate>]`

## Step 3 — GENERATE

Create the `.codestudio/` directory with all harness files.

### Read templates from:
```
./templates/
```

### Files to generate:

#### Always generate:

1. **`.codestudio/task.py`** — from `task.py.tmpl`
   Copy as-is. This is the task manager script.

2. **`.codestudio/agents/orchestrator.agent.md`** — from `orchestrator.agent.md.tmpl`
   This is the loop protocol as a custom agent. Replace `{{PROJECT_NAME}}`.
   Add security and UI notes if flags were detected (see detection-rules.md).
   
   **Does NOT touch any existing `codestudio-instructions.md`.** The harness uses the agents/ folder,
   so projects with their own workspace instructions are unaffected.

3. **`.codestudio/instructions/task-conventions.instructions.md`** — from `task-conventions.instructions.md.tmpl`
   Replace `{{PROJECT_NAME}}`. Loaded on-demand when working on tasks.

**Verify required skills exist.** Check that all mandatory skills are present at `~/.agents/skills/<name>/SKILL.md`:
   - `spec-driven-development`, `planning-and-task-breakdown`, `test-driven-development`
   - `incremental-implementation`, `debugging-and-error-recovery`, `code-review-and-quality`
   - `security-and-hardening`, `git-workflow-and-versioning`
   - `frontend-ui-engineering` (if UI_PROJECT is true)
   
   If any required skill is missing, warn the user:
   > "⚠️ Missing required skills: [list]. The harness expects these for full loop operation."

4. **`.codestudio/tasks/index.json`** — from `index.json.tmpl`
   Empty array for new projects. **Preserve existing if upgrade mode.**

5. **`.codestudio/progress.md`** — from `progress.md.tmpl`
   Replace `{{PROJECT_NAME}}`. **Preserve existing if upgrade mode.**

6. **`.codestudio/project-context.md`** — from `project-context.md.tmpl`
   Replace template variables:
   - `{{PROJECT_NAME}}` → project name
   - `{{STACK}}` → detected/provided tech stack (e.g., "Next.js 14, TypeScript, Prisma, PostgreSQL")
   - `{{ARCHITECTURE}}` → detected/provided structure (e.g., "API routes → Services → Repositories")
   - `{{CONVENTIONS}}` → detected/provided patterns (e.g., "kebab-case files, Zod validation at boundaries")
   - `{{BOUNDARIES}}` → detected/provided constraints (e.g., "Don't modify: migrations/, .env")
   - `{{TIER}}` → L0/L1/L2/L3 based on test reality
   - `{{TIER_DESCRIPTION}}` → what the tier proves and does NOT prove
   - `{{GATES}}` → numbered evidence gates with command, threshold, and ratchet flag
   
   Example GATES for a Next.js + TypeScript + Jest project at L1:
   ```
   1. **build**: `npm run build` — threshold: exit code 0
   2. **lint**: `npx eslint .` — threshold: exit code 0
   3. **typecheck**: `npx tsc --noEmit` — threshold: exit code 0
   4. **test**: `npx jest` — threshold: exit code 0 (ratchet)
   5. **coverage-diff**: `npx c8 --reporter=lcov npx jest -- --changedSince=HEAD~1` — threshold: diff line rate ≥ 0.80 (ratchet)
   ```
   
   For existing repos: populate from scan findings (source files, directory structure, configs).
   For empty repos: populate from interview answers.
   **Preserve existing if upgrade mode** — user may have customized it.

### Upgrade mode behavior:

| File | Action |
|------|--------|
| `task.py` | Overwrite (latest version) |
| `agents/orchestrator.agent.md` | Overwrite (re-detect settings) |
| `instructions/task-conventions.instructions.md` | Overwrite |
| `project-context.md` | **PRESERVE** (user may have customized) |
| `tasks/index.json` | **PRESERVE** |
| `tasks/*.md` | **PRESERVE** |
| `progress.md` | **PRESERVE** |
| `archive/` | **PRESERVE** |

## Step 4 — SEED TASKS

**For empty repos**: Generate 3-5 initial tasks from interview answers.
Example for a new web app:
```
python3 .codestudio/task.py add "Initialize project with [framework]"
python3 .codestudio/task.py add "Set up dev environment and CI" --needs T-001
python3 .codestudio/task.py add "Implement [first feature]" --needs T-001
python3 .codestudio/task.py add "Add authentication" --needs T-001 --backlog
```

**For existing repos with issues/TODO**: Scan for TODO comments, open issues, or README roadmap items and convert to tasks.

**For upgrades**: Do not add tasks. Existing task list is authoritative.

## Step 5 — VERIFY HARNESS

Run a smoke test:
```bash
python3 .codestudio/task.py status
```

Expected output: PROJECT STATUS with counts. If tasks were seeded, they should appear.

Also verify:
- `project-context.md` has correct verification steps
- `task.py` is executable (suggest `chmod +x .codestudio/task.py` on Unix)
- `.codestudio/` is in `.gitignore` (or not, depending on team preference — ask user)

## Output

After generation, report:
```
Harness created at .codestudio/
  ├── task.py                    — task manager (verify, done, block, etc.)
  ├── agents/
  │   └── orchestrator.agent.md  — loop protocol (@orchestrator)
  ├── instructions/
  │   └── task-conventions.instructions.md — task file format, commits
  ├── catalogs/
  │   └── defects-{{FRAMEWORK}}.md — framework-specific defect catalog
  ├── project-context.md         — stack, architecture, tier, gates
  ├── harness-lock.json          — ratchet floors (thresholds only go up)
  ├── tasks/index.json           — task index (N tasks seeded)
  ├── evidence/                  — gate output per task (gitignored)
  └── progress.md                — session memory

Detected: [language], verification steps: [list]
Skills available: [list of installed skills]

To start working: Invoke @orchestrator or run `python3 .codestudio/task.py next`
```

## Reference Documents

For detailed information on specific aspects:

- **Loop protocol**: `./references/loop-protocol.md` — how the 9-stage loop works, task state machine, constraints
- **Stages catalog**: `./references/stages-catalog.md` — which SDLC skills map to which stages
- **Detection rules**: `./references/detection-rules.md` — how repo scanning works, signal-to-config mapping
