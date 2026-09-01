# Stages Catalog — Skill Mapping

Each loop stage has **mandatory** skills that the agent MUST read and follow. Skills are not optional enhancers — they are required process steps. The agent reads each skill's `SKILL.md` at the corresponding stage and follows its instructions.

Skills live at `~/.agents/skills/<name>/SKILL.md`.

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

The `orchestrator.agent.md` template names each skill at the relevant stage with its full path:

```markdown
### 4. BUILD
...
Read and follow `~/.agents/skills/test-driven-development/SKILL.md`: RED → GREEN → REFACTOR.
```

The agent reads the skill file using `read_file` and follows its instructions. Skills are mandatory — the agent must not skip them or rely on prior knowledge of their content.

## Skill Detection

During harness generation, the skill detects installed skills by checking:
```
~/.agents/skills/<name>/SKILL.md
```

This determines:
1. Which skill references to include in `orchestrator.agent.md`
2. Which context hints to add in task file templates

## Without Skills Installed

If a skill file does not exist at the expected path, the agent falls back to the inline guidance in `orchestrator.agent.md` for that stage. However, this is a degraded mode — the harness-creator should ensure all required skills are present during generation (Step 2 — INTERVIEW) and warn the user if any are missing.
