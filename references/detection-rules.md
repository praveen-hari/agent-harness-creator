# Detection Rules

When the harness-creator skill runs on a repository, it scans for project signals to auto-configure the harness. Each rule maps a file or pattern to a configuration decision.

## Language / Framework Detection

| Signal | Detected As | Default verify_cmd |
|--------|------------|-------------------|
| `package.json` | Node.js | `npm test` |
| `package.json` + `next.config.*` | Next.js | `npm test` |
| `package.json` + `vite.config.*` | Vite | `npm test` |
| `requirements.txt` or `pyproject.toml` or `setup.py` | Python | `pytest` |
| `Cargo.toml` | Rust | `cargo test` |
| `*.csproj` or `*.sln` | .NET | `dotnet test` |
| `go.mod` | Go | `go test ./...` |
| `Makefile` | Make-based | `make test` |
| `build.gradle` or `pom.xml` | Java/Kotlin | `./gradlew test` or `mvn test` |
| `mix.exs` | Elixir | `mix test` |
| `Gemfile` | Ruby | `bundle exec rake test` |
| `composer.json` | PHP | `composer test` |

If multiple signals exist (e.g., monorepo), prefer the root-level `package.json` or the primary build file.

If no recognizable signal: prompt the user for verify_cmd during INTERVIEW.

## Project Structure Signals

| Signal | Configuration Effect |
|--------|---------------------|
| `.git` directory exists | COMMIT stage enabled. Use `git-workflow-and-versioning`. |
| No `.git` | Skip COMMIT stage. Note in instructions: "Initialize git when ready." |
| `src/` or `lib/` present | Standard project. Auto-detect patterns. |
| No source files at all | **Empty repo** → enter INTERVIEW mode. Use `interview-me` skill. |
| > 10 source files | Generate `reviewer.agent.md` for independent review. |
| ≤ 10 source files | Skip reviewer agent. Self-review in loop is sufficient. |

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

| Signal | Detected Framework | verify_cmd Override |
|--------|-------------------|-------------------|
| `jest.config.*` or `"jest"` in package.json | Jest | `npx jest` |
| `vitest.config.*` | Vitest | `npx vitest run` |
| `pytest.ini` or `conftest.py` | pytest | `pytest` |
| `*.test.rs` or `#[cfg(test)]` | Rust tests | `cargo test` |
| `*_test.go` | Go tests | `go test ./...` |
| `*.spec.ts` + `cypress.config.*` | Cypress | `npx cypress run` |
| `playwright.config.*` | Playwright | `npx playwright test` |

## Config Stacking

Detection is additive. A Next.js project with JWT auth and Playwright tests produces:
- verify_cmd: `npm test && npx playwright test`
- SECURITY flag with auth note
- UI flag with frontend-ui-engineering reference
- reviewer.agent.md generated (if >10 files)

## Empty Repository

When no source files are detected:
1. Enter INTERVIEW mode
2. Use `interview-me` skill to gather: what, for whom, tech stack, first feature
3. Generate initial tasks from interview answers
4. Set verify_cmd based on chosen tech stack
5. Proceed with normal loop

## Upgrade Detection

When `.codestudio/` already exists:
1. Preserve: `tasks/`, `progress.md`, `archive/`
2. Update: `task.py`, `codestudio-instructions.md`, `reviewer.agent.md`
3. Report: what was updated vs preserved
