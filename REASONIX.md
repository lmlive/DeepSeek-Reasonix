# Reasonix project memory

This file is loaded into every session's system prompt (the cache-stable prefix),
so keep it concise and durable — it is the project's standing instructions to the
agent. It is the Reasonix analog of Claude Code's CLAUDE.md.

## Project

Reasonix is a **config- and plugin-driven AI coding agent** — a single static Go binary
tuned around DeepSeek's prefix cache for low token costs across long sessions.
Module `reasonix` (Go 1.25+). Entry points:
- **CLI**: `cmd/reasonix/main.go` → `cli.Run()` → chat TUI (`charm.land/bubbletea/v2`).
- **Desktop**: `desktop/` — separate Go module (Wails/webview), shares the `internal/` kernel.
- **HTTP/SSE server**: `internal/serve/` — web frontend for remote/SSH access.

Configuration via `reasonix.toml`; providers, tools, and plugins are all declarative.

## Commands

```bash
make build             # CLI + example plugin → bin/
make cross             # cross-compile to 6 platforms → dist/
make test              # go test ./...
make vet               # go vet ./...
make fmt               # gofmt -w .
make desktop-test      # cd desktop && go test .
make hooks             # install git pre-push hook (go vet)
make clean             # rm -rf bin dist
```

CI: GitHub Actions (`ci.yml`) — test matrix across ubuntu/macos/windows,
`golangci-lint` (errcheck, govet, ineffassign, staticcheck, unused).

## Architecture

Six load-bearing layers:

1. **`internal/boot`** — Assembly layer. Loads config, resolves models, builds tool
   registry (built-ins + plugins), wires permissions, and produces a `control.Controller`.
   Every frontend shares this single bootstrap path.
2. **`internal/control`** — Transport-agnostic session driver. Owns the agent run loop,
   turn lifecycle, approvals, cancellation. Emits typed events to an `event.Sink`.
   Central file: `controller.go` (200+ KB — the orchestrator).
3. **`internal/agent`** — Agent logic. LLM interaction, tool execution, session save/load,
   compaction, subagent management, fleet dispatch, evidence tracking, planning mode.
   Core: `agent.go`, `coordinator.go` (two-model planner+executor), `task.go` (subagents).
4. **`internal/provider`** — LLM provider abstraction. Schema canonicalization, retry logic,
   dialect handling. Subpackages: `openai/`, `anthropic/` (blank-import wired in main).
5. **`internal/tool`** — Tool framework. Registry, contract validation, progress reporting.
   `tool/builtin/` auto-registers at compile time via blank import.
6. **`internal/cli`** — CLI frontend. Chat TUI, command parsing, MCP manager, setup wizard,
   theme, transcription. The largest UI surface.

Supporting packages (selected):
- **`internal/config`** — TOML config parsing and resolution.
- **`internal/memory`** — Persistent project memory (frontmatter files + MEMORY.md).
- **`internal/skill`** — Inline and subagent skill playbooks.
- **`internal/acp`** — Agent Communication Protocol implementation.
- **`internal/sandbox`** — Execution sandbox for tools.
- **`internal/plugin`** — External plugin lifecycle (stdio JSON-RPC / MCP-compatible).
- **`desktop/`** — Wails desktop app (separate module, CGO-enabled).

## Conventions

- Go kernel under `internal/`; each package owns one concern and documents it in a
  package comment. Match the surrounding comment density and idiom when editing.
- One transport-agnostic `control.Controller` sits behind every frontend (chat
  TUI, HTTP/SSE serve, Wails desktop). Add behavior to the controller, not a
  frontend, so all three inherit it.
- Cache-first: the system-prompt prefix (base prompt + tools + memory) must stay
  byte-stable across turns so DeepSeek's automatic prefix cache stays warm. Never
  mutate it mid-session — ride the turn tail instead (see `control.Compose`).

## Memory

- Standing instructions are hierarchical: committed/shared `REASONIX.md`,
  `AGENTS.md`, and `CLAUDE.md`; personal `*.local.md` variants; matching files in
  ancestor directories; and user-global files under the memory state root
  (`REASONIX_STATE_HOME`, otherwise `REASONIX_HOME`, otherwise `~/.reasonix` on
  macOS/Linux or `%APPDATA%\reasonix` on Windows). All distinct supported files
  in a directory load; `AGENTS.md` is not merely a fallback.
- `@path` on its own line imports another file's contents.
- `#<note>` in chat quick-adds an always-on instruction. The `remember` tool
  instead saves a fallible background fact (frontmatter file + `MEMORY.md`
  index). Fact `type` classifies content; independent `scope` controls whether it
  is project-only (the default) or explicitly global. The index loads into the
  stable prefix on the next session; global user/feedback bodies also load as
  lower-priority compatibility guidance. The current turn receives a tail note.

## Notes

## Pre-push CI simulation

Run these **before every commit** to catch the fastest CI failures locally:

```bash
gofmt -w .                          # catches gofmt (saves ~13s CI)
go vet ./...                        # catches vet warnings (saves ~52s CI/lint)
go test ./internal/tool/builtin/ ./internal/boot/  # catches tool/boot test breaks
```

CI runs `golangci-lint` (not locally available), but gofmt + vet already block ~80% of fast-fail scenarios.

## Import cycle rule

Before importing a new internal package from a non-test file, verify the target package's **test files** aren't already importing back to you:

```
# BAD: agent(_test.go) → tool/builtin(sessions.go) → agent  → setup failed
```

Use `go test ./path/to/target/` to detect cycles **before** pushing. A `[setup failed]` message means a cycle exists.

## PR hygiene

- **One force-push per round of review feedback.** Multiple force-pushes destroy review history and confuse reviewers.
- **Keep the PR diff minimal.** Only the files relevant to the PR's purpose — no stray changes from other branches.
- **Amend, don't add commits, for review feedback** — keeps the commit history clean.

## Cache-impact PR metadata

When PR changes touch files under `internal/boot/`, `internal/tool/`, `internal/provider/`, or other cache-sensitive paths (listed in `scripts/check-cache-impact.sh`), the PR body MUST include these lines at the end:

```
Cache-impact: <none|low|medium|high> — <reason>
Cache-guard: <focused guard test/command or existing guard rationale>
```

If the PR also touches files under `internal/config/`, `internal/memory/`, `internal/outputstyle/`, `internal/skill/`, or `internal/boot/`, add:

```
System-prompt-review: <reviewer/approval note>
```

Values `n/a`, `none`, `todo`, `tbd` are rejected — use a descriptive reason instead.
