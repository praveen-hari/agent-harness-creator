# Detection Rules

When the harness-creator skill runs on a repository, it scans for project signals to auto-configure the harness. Each rule maps a file or pattern to a configuration decision.

## Language / Framework Detection

| Signal | Detected As | Default Verify Steps |
|--------|------------|---------------------|
| `package.json` | Node.js | lint → test |
| `package.json` + `next.config.*` | Next.js | lint → typecheck → test → build |
| `package.json` + `vite.config.*` | Vite | lint → typecheck → test |
| `requirements.txt` or `pyproject.toml` or `setup.py` | Python | lint → test |
| `Cargo.toml` | Rust | clippy → test |
| `*.csproj` or `*.sln` | .NET | build → test |
| `go.mod` | Go | vet → test |
| `Makefile` | Make-based | test |
| `build.gradle` or `pom.xml` | Java/Kotlin | test |
| `mix.exs` | Elixir | test |
| `Gemfile` | Ruby | test |
| `composer.json` | PHP | test |

If multiple signals exist (e.g., monorepo), prefer the root-level `package.json` or the primary build file.

If no recognizable signal: prompt the user for verification steps during INTERVIEW.

### Default Verify Steps by Stack

Each detected stack produces a numbered list of verification steps for `project-context.md`:

**Node.js / Next.js / Vite (TypeScript)**:
```
1. **Lint**: `npm run lint` — code style and static analysis
2. **Typecheck**: `npx tsc --noEmit` — type safety
3. **Test**: `npm test` — unit and integration tests
4. **Build**: `npm run build` — compilation succeeds
```

**Python**:
```
1. **Lint**: `ruff check .` (or `flake8`) — code style
2. **Typecheck**: `mypy .` — if mypy is in dependencies
3. **Test**: `pytest` — unit and integration tests
```

**Rust**:
```
1. **Lint**: `cargo clippy -- -D warnings` — lints as errors
2. **Test**: `cargo test` — all tests pass
```

**.NET**:
```
1. **Build**: `dotnet build` — compilation succeeds
2. **Test**: `dotnet test` — all tests pass
3. **Format**: `dotnet format --verify-no-changes` — if dotnet-format is configured
```

**Go**:
```
1. **Vet**: `go vet ./...` — static analysis
2. **Test**: `go test ./...` — all tests pass
```

Only include steps for tools that are actually detected in the project (e.g., skip Typecheck if no `tsconfig.json`, skip Lint if no linter config).

## Project Structure Signals

| Signal | Configuration Effect |
|--------|---------------------|
| `.git` directory exists | COMMIT stage enabled. Use `git-workflow-and-versioning`. |
| No `.git` | Skip COMMIT stage. Note in instructions: "Initialize git when ready." |
| `src/` or `lib/` present | Standard project. Auto-detect patterns. |
| No source files at all | **Empty repo** → enter INTERVIEW mode. Use `interview-me` skill. |
| > 10 source files | Complex project — ensure project-context.md captures architecture. |
| ≤ 10 source files | Small project — lightweight project-context.md is sufficient. |

## Security Signals

Scan source files for patterns that indicate security-sensitive code:

| Pattern | Effect |
|---------|--------|
| `auth`, `login`, `password`, `session` in filenames or imports | Flag SECURITY in task instructions |
| `jwt`, `token`, `oauth` in dependencies | Flag SECURITY |
| `crypto`, `encrypt`, `hash` in source | Flag SECURITY |
| `.env` file present | Note: "Secrets management in use. Verify .gitignore." |
| `api/`, `routes/`, `endpoints/` directories | Flag: "API endpoints → validate input at boundaries." |

Security flags add this to `codestudio-instructions.md`:
```
Security Note: This project handles [auth/tokens/encryption]. The `security-and-hardening` skill should be used during REVIEW for any task that touches these areas.
```

## UI Detection

| Signal | Effect |
|--------|--------|
| `*.tsx`, `*.jsx`, `*.vue`, `*.svelte` files | Flag UI project |
| `components/`, `pages/`, `views/` directories | Flag UI structure |
| CSS/SCSS/Tailwind config present | Note styling framework |

UI flags add this to `codestudio-instructions.md`:
```
UI Note: This project has frontend components. Use `frontend-ui-engineering` skill for any task touching UI.
```

## Test Framework Detection

| Signal | Detected Framework | Test Step Command |
|--------|-------------------|------------------|
| `jest.config.*` or `"jest"` in package.json | Jest | `npx jest` |
| `vitest.config.*` | Vitest | `npx vitest run` |
| `pytest.ini` or `conftest.py` | pytest | `pytest` |
| `*.test.rs` or `#[cfg(test)]` | Rust tests | `cargo test` |
| `*_test.go` | Go tests | `go test ./...` |
| `*.spec.ts` + `cypress.config.*` | Cypress (E2E) | `npx cypress run` |
| `playwright.config.*` | Playwright (E2E) | `npx playwright test` |

E2E test frameworks add an additional verification step after unit tests:
```
N. **E2E Tests**: `npx playwright test` — end-to-end tests
```

## Lint / Format Detection

| Signal | Lint Step Command |
|--------|------------------|
| `.eslintrc.*` or `eslint.config.*` or `"eslint"` in package.json | `npx eslint .` |
| `biome.json` | `npx biome check .` |
| `.prettierrc.*` | `npx prettier --check .` (format step) |
| `ruff.toml` or `[tool.ruff]` in pyproject.toml | `ruff check .` |
| `.flake8` or `[flake8]` in setup.cfg | `flake8` |
| `clippy` in Cargo deps | `cargo clippy -- -D warnings` |
| `.golangci.yml` | `golangci-lint run` |

## Config Stacking

Detection is additive. A Next.js project with JWT auth and Playwright tests produces:
- Verify steps:
  ```
  1. **Lint**: `npx eslint .` — code style
  2. **Typecheck**: `npx tsc --noEmit` — type safety
  3. **Test**: `npx jest` — unit tests
  4. **E2E Tests**: `npx playwright test` — end-to-end
  5. **Build**: `npm run build` — compilation
  ```
- SECURITY flag with auth note
- UI flag with frontend-ui-engineering reference


## Empty Repository

When no source files are detected:
1. Enter INTERVIEW mode
2. Use `interview-me` skill to gather: what, for whom, tech stack, first feature
3. Generate initial tasks from interview answers
4. Set verification steps based on chosen tech stack
5. Proceed with normal loop

## Upgrade Detection

When `.codestudio/` already exists:
1. Preserve: `tasks/`, `progress.md`, `archive/`
2. Update: `task.py`, `codestudio-instructions.md`
3. Report: what was updated vs preserved
