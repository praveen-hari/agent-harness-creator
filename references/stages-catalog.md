# Stages Catalog — Skill Mapping

Each loop stage can optionally leverage an installed SDLC skill. Skills are enhancers — if not installed, the agent follows inline guidance from `codestudio-instructions.md`. If installed, the agent reads the skill's `SKILL.md` for deeper process guidance.

Skills live in `~/.agents/skills/<name>/SKILL.md`.

## Stage → Skill Mapping

| Stage | Primary Skill | Secondary Skill | When to Use |
|-------|--------------|-----------------|-------------|
| PICK | — | — | Always (script-driven) |
| SPEC | `spec-driven-development` | — | Complex or unclear tasks |
| PLAN | `planning-and-task-breakdown` | — | Multi-step tasks |
| BUILD | `incremental-implementation` | `test-driven-development` | Always |
| VERIFY | `debugging-and-error-recovery` | — | On failure (up to 3 retries) |
| REVIEW | `code-review-and-quality` | `security-and-hardening` | >10 files changed or sensitive task |
| COMMIT | `git-workflow-and-versioning` | — | Always (if .git exists) |
| LEARN | — | — | Always |

## Context-Specific Skills

These activate based on what the task touches, not the loop stage:

| Skill | Trigger |
|-------|---------|
| `frontend-ui-engineering` | Task involves UI components, pages, or layouts |
| `frontend-design-system` | Task involves design tokens, themes, or style systems |
| `context-engineering` | Task involves configuring agent rules or context |
| `documentation-and-adrs` | Task involves architectural decisions or public API changes |
| `shipping-and-launch` | Task involves deployment, rollout, or production readiness |

## How Skills Are Invoked

The `codestudio-instructions.md` template names each skill at the relevant stage:

```markdown
### 4. BUILD
...
If the `test-driven-development` skill is available, follow it: RED → GREEN → REFACTOR.
```

The agent checks if the skill file exists. If it does, the agent reads it and follows its instructions. If not, the inline guidance in the instructions file is sufficient.

## Skill Detection

During harness generation, the skill detects installed skills by checking:
```
~/.agents/skills/<name>/SKILL.md
```

This determines:
1. Which skill references to include in `codestudio-instructions.md`
2. Whether to generate the `reviewer.agent.md` (needs `code-review-and-quality`)
3. Which context hints to add in task file templates

## Without Any Skills

The harness works standalone. The loop protocol provides enough inline guidance for:
- Basic spec writing (describe what and why)
- Simple planning (checkbox subtasks)
- Standard building (one change at a time)
- Verification (run tests, fix failures)
- Self-review (5-point checklist)
- Conventional commits
- Learning (log decisions)

Skills upgrade the quality of each stage, but are never required.
