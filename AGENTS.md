# Crush Development Guide

## Project Overview

Crush is a terminal-based AI coding assistant written in Go. The CLI entrypoint
is `main.go`, which boots Cobra commands from `internal/cmd`. The app combines
provider/model configuration, session persistence, tool execution, Bubble Tea
UI, LSP integration, MCP integration, and SQLite-backed history.

- Module path: `github.com/charmbracelet/crush`
- Go version: `1.26.1` (`go.mod`)
- Primary config/task runner: `Taskfile.yaml`
- Main local config file used in this repo: `crush.json`

## Before You Change Anything

- Read the nearest `AGENTS.md` before touching that area:
  - Root guide: `AGENTS.md`
  - TUI-specific guide: `internal/ui/AGENTS.md`
  - Stats web assets: `internal/cmd/stats/AGENTS.md`
- This repository has no `.cursor/rules`, `.cursorrules`, or Copilot instruction
  files in the checked tree.
- The codebase itself reads context files from the working directory:
  `AGENTS.md`, `CRUSH.md`, `CLAUDE.md`, `GEMINI.md`, plus `.local` variants.

## Essential Commands

All commands below are observed from `Taskfile.yaml`, CI workflows, or project
config.

### Build and Run

- Build binary: `task build`
- Build directly: `go build .`
- Build all packages with race detector in CI style: `go build -race ./...`
- Run app after building: `task run`
- Run directly: `go run .`
- Run with profiling enabled: `task dev`
- Install locally with version ldflags: `task install`

### Tests

- Full test suite: `task test`
- Direct equivalent: `go test -race -failfast ./...`
- Run a single package/test: `go test ./internal/config -run TestConfig_LoadFromBytes`
- Re-record agent VCR cassettes: `task test:record`
- Update golden/snapshot-style outputs when tests support it: `go test ./... -update`

### Lint / Format / Modernize

- Lint: `task lint`
- Lint and auto-fix: `task lint:fix`
- Log capitalization check only: `task lint:log`
- Format Go: `task fmt`
- Format stats HTML/CSS/JS: `task fmt:html`
- Modernize Go code: `task modernize`

### Code Generation / Project Maintenance

- Regenerate config schema: `task schema`
- Regenerate Hyper embedded provider data: `task hyper`
- Regenerate SQLC output: `task sqlc`
- Update key deps and tidy module: `task deps`

### Profiling

When `CRUSH_PROFILE=true`, `main.go` exposes pprof on `localhost:6060`.
Observed helper tasks:

- `task profile:cpu`
- `task profile:heap`
- `task profile:allocs`

## CI and Release Expectations

Observed from `.github/workflows/*.yml` and `.goreleaser.yml`:

- CI build runs on Ubuntu, macOS, and Windows.
- CI verifies:
  - `go mod tidy`
  - clean diff after tidy
  - `go build -race ./...`
  - `go test -race -failfast ./...`
- Lint uses shared Charm workflow with `.golangci.yml`.
- Security workflows run CodeQL, Grype, `govulncheck`, and dependency review.
- Schema/hyper artifacts are auto-updated on `main` when config-related files
  change.
- Goreleaser builds with:
  - `CGO_ENABLED=0`
  - `GOEXPERIMENT=greenteagc`
- Release packaging generates shell completions and manpages before archiving.

If you touch dependency graphs, schema generation, or embedded provider data,
expect CI or automation to care.

## Repository Layout

High-level structure observed from the tree and representative files:

```text
main.go                     CLI entrypoint
Taskfile.yaml               Main task runner
crush.json                  Repo-local config for Crush itself
schema.json                 Generated JSON schema output
sqlc.yaml                   SQLC configuration
internal/
  app/                      Top-level application wiring and lifecycle
  agent/                    Agent orchestration, prompts, tools, MCP, providers
  cmd/                      Cobra commands and subcommands
  config/                   Config loading, defaults, provider/model resolution
  db/                       SQLite access, SQLC output, migrations, raw SQL
  event/                    Telemetry/event plumbing
  filetracker/              Tracks files read during sessions
  history/                  Prompt/file history services
  lsp/                      LSP manager and clients
  message/                  Message/content models
  permission/               Tool permission requests and allow-lists
  projects/                 Project registration/lookup
  pubsub/                   Internal event broker
  session/                  Session persistence
  shell/                    Bash execution and background jobs
  skills/                   Skill discovery/loading
  ui/                       Bubble Tea / Ultraviolet TUI
  update/                   Update checks
```

Database-specific layout:

- Migrations: `internal/db/migrations/`
- Raw SQL: `internal/db/sql/`
- Generated SQLC code: `internal/db/`

Observed migration history includes session/message storage plus later additions
such as summary message IDs, provider fields, todos, and read-files tracking.

## Architecture Notes

### App Boot Flow

- `main.go` optionally starts pprof when `CRUSH_PROFILE` is set, then calls
  `cmd.Execute()`.
- `internal/cmd/root.go` resolves working dir, loads config, creates the app,
  connects SQLite, registers the project, and launches either TUI or
  non-interactive flows.
- `internal/app/app.go` wires together sessions, messages, history,
  permissions, file tracking, LSP manager, agent coordinator, MCP startup, and
  update checks.

### Agent Layer

- `internal/agent/agent.go` is the core session-based orchestration layer.
- Providers are served through `charm.land/fantasy`.
- The agent attaches MCP server instructions into the system prompt when MCP
  clients are connected.
- Sessions queue prompts when already busy.
- Title generation is triggered for first-message sessions.
- Tools are configured centrally and injected into agents.

### Configuration Model

Observed from `README.md`, `internal/config/load.go`, and `crush.json`:

- Config precedence documented in the README:
  1. `.crush.json`
  2. `crush.json`
  3. `$HOME/.config/crush/crush.json`
- Workspace data is also merged from a generated workspace config inside the
  data directory (`<data-dir>/crush.json`) with highest priority.
- `config.Load` applies defaults, sets up logging, loads providers, configures
  selected models, and sets up agents.
- If the working directory is not inside a git worktree, Crush reduces file-walk
  depth/item limits.
- Apple Terminal gets transparent mode enabled automatically.
- This repo’s `crush.json` configures `gopls` with `gofumpt`, code lenses,
  staticcheck, semantic tokens, and analysis settings.

## UI / TUI Guidance

If you touch anything under `internal/ui/`, read `internal/ui/AGENTS.md` first.
Key points already documented there and confirmed by structure:

- The top-level UI model is centralized in `internal/ui/model/ui.go`.
- UI rendering uses a hybrid approach:
  - Ultraviolet screen-buffer drawing at the top level
  - string-rendered subcomponents painted onto the screen
- Subcomponents are generally imperative/stateful helpers, not full nested
  Bubble Tea models.
- Keep expensive work out of `Update`; return `tea.Cmd` for side effects.
- Use `github.com/charmbracelet/x/ansi` for ANSI-aware string handling.

Stats-specific note:

- `internal/cmd/stats/AGENTS.md` only specifies one rule: format CSS/HTML/JS
  with `prettier`.
- Relevant files are `internal/cmd/stats/index.html`, `index.css`, and
  `index.js`.

## Testing Patterns

Observed from representative tests and task configuration:

- The project uses `testing` plus `github.com/stretchr/testify/require` and
  `assert`.
- Parallel tests are common (`t.Parallel()`).
- Temporary directories are used heavily (`t.TempDir()`).
- Some tests use `testing/synctest` and `go.uber.org/goleak` for concurrency
  validation.
- Agent integration tests in `internal/agent/agent_test.go` use
  `charm.land/x/vcr` recorders and are skipped on Windows in at least one case.
- Re-recording VCR fixtures is handled by `task test:record`, which removes
  `internal/agent/testdata` and reruns the agent tests.

When changing tests or behavior that affects snapshots/cassettes, check whether
VCR recordings or golden outputs need regeneration.

## Code Style and Conventions

Observed from `.golangci.yml`, existing code, tests, and current AGENTS content:

### Go Style

- Format Go with `gofumpt` (`task fmt`).
- Imports are goimports-style grouped.
- Exported names use PascalCase; unexported names use camelCase.
- `context.Context` is commonly the first parameter for operations.
- Errors are wrapped with `fmt.Errorf(... %w ...)`.
- JSON tags use `snake_case`.
- File permissions use octal literals such as `0o644` and `0o755`.

### Logging

- Log messages must start with a capital letter.
- This is explicitly checked by `task lint:log` via
  `scripts/check_log_capitalization.sh`.

### Comments

- Existing project guidance says standalone comments should start with a capital
  letter and end with a period.

### Linters Enabled

Observed from `.golangci.yml`:

- `bodyclose`
- `goprintffuncname`
- `misspell`
- `noctx`
- `nolintlint`
- `rowserrcheck`
- `sqlclosecheck`
- `staticcheck`
- `tparallel`
- `whitespace`

Formatters enabled via golangci-lint:

- `gofumpt`
- `goimports`

## Tooling and Non-Obvious Patterns

### SQLC / Database

- SQLC is configured in `sqlc.yaml` for SQLite.
- Source-of-truth SQL lives in `internal/db/sql/` and migrations in
  `internal/db/migrations/`.
- Generated code is emitted into `internal/db/` with interfaces and prepared
  queries enabled.
- If you change SQL or migrations, run `task sqlc`.

### Release Artifacts

Observed from `.goreleaser.yml`:

- Release hooks generate:
  - bash/zsh/fish completions
  - gzipped manpages
- Packaging includes `README*`, `LICENSE*`, `manpages/*`, and `completions/*`.
- Build metadata injects `internal/version.Version` via ldflags.

### Runtime / Env Details

- `.env` files are auto-loaded in `main.go` via `github.com/joho/godotenv/autoload`.
- `CRUSH_PROFILE=true` enables pprof.
- `CRUSH_DISABLE_METRICS` and `DO_NOT_TRACK` are checked when deciding whether
  to initialize metrics.
- Root command supports `--cwd`, `--data-dir`, `--debug`, `--yolo`,
  `--session`, and `--continue`.

### Permissions / File Access

Observed from `internal/agent/tools/view.go`:

- Tool implementations resolve relative paths against the working directory.
- Access outside the working directory may require a permission request.
- Skill files are treated specially.
- The `view` tool records files read and waits briefly for LSP diagnostics after
  opening files.

## Practical Advice for Future Agents

- Prefer `task` commands when available; they encode the repo’s intended env and
  flags.
- After changing Go code, a good default verification path is:
  1. `task fmt`
  2. targeted `go test` package(s)
  3. `task test` if the change is broad
- If you touch TUI code, read `internal/ui/AGENTS.md` first.
- If you touch stats frontend assets, run `task fmt:html`.
- If you touch config schema or Hyper provider generation inputs, run the
  matching generation task (`task schema`, `task hyper`).
- If you touch SQL, regenerate SQLC output with `task sqlc`.
- If CI would run `go mod tidy` for your change, make sure it leaves no diff.

## Observed Gaps / Things Not Present

Only include what is actually in the repo:

- No `Makefile`
- No root `package.json`
- No checked-in `.cursor/rules/*.md`
- No checked-in `.cursorrules`
- No checked-in `.github/copilot-instructions.md`
