---
name: kimi-code-cli
description: Delegate coding tasks to Kimi Code CLI (Moonshot AI's terminal coding agent). Use for feature work, refactors, PR review, batch scripts, and iterative sessions. Requires the `kimi` CLI installed and logged in.
version: 1.0.0
author: [Andy Liu](https://github.com/ascl1u)
license: MIT
metadata:
  hermes:
    tags: [Coding-Agent, Kimi, Moonshot, Code-Review, Refactoring, ACP]
    related_skills: [claude-code, codex, opencode, hermes-agent]
---

# Kimi Code CLI

Use [Kimi Code CLI](https://moonshotai.github.io/kimi-cli/en/guides/getting-started.html) as an autonomous coding worker orchestrated by Hermes terminal/process tools. Kimi Code CLI is Moonshot AI's terminal coding agent with a TUI and CLI, plus optional browser UI (`kimi web`) and IDE integration (`kimi acp`).

## When to Use

- User explicitly asks to use Kimi Code CLI
- You want an external coding agent to implement/refactor/review code
- You need long-running coding sessions with progress checks (including Ralph Loop)
- You want parallel task execution in isolated workdirs/worktrees
- You need plan-then-implement workflows (plan mode before edits)

## Prerequisites

- Kimi Code CLI installed: `curl -LsSf https://code.kimi.com/install.sh | bash` (Windows: `Invoke-RestMethod https://code.kimi.com/install.ps1 | Invoke-Expression`), or with `uv` already on PATH: `uv tool install --python 3.13 kimi-cli`
- Auth configured: `kimi login` (browser OAuth) or `/login` on first interactive launch
- Verify: `kimi --version` and `kimi info`
- Git repository for code tasks (recommended)
- `pty=true` for interactive TUI sessions

## Two Orchestration Modes

Hermes interacts with Kimi Code CLI in two fundamentally different ways. Choose based on the task.

### Mode 1: Print Mode (`--print` / `-p`) — Non-Interactive (PREFERRED for most tasks)

Non-interactive: reads the prompt, runs the agent loop, exits on completion. `--print` **implicitly enables `--yolo`**, so all tool calls auto-approve. No PTY needed.

```
terminal(command="kimi --print -p 'Add retry logic to API calls and update tests'", workdir="~/project", timeout=180)
```

### Mode 2: Interactive TUI (background + PTY) — Multi-Turn Sessions

For iterative, multi-turn work. Start `kimi` in the background with a PTY, then drive it with the `process` tool.

```
terminal(command="kimi", workdir="~/project", background=true, pty=true)
# returns session_id
process(action="submit", session_id="<id>", data="Implement OAuth refresh flow and add tests")
process(action="poll", session_id="<id>")
process(action="log", session_id="<id>")
```

## One-Shot Tasks (Print Mode)

```
# Inline prompt
terminal(command="kimi --print -p 'Add dark mode toggle to settings'", workdir="~/project")

# Only the final assistant message (great for commit messages)
terminal(command="kimi --quiet -p 'Write a Git commit message for the current changes'", workdir="~/project")

# Pipe context via stdin
terminal(command="git diff origin/main...HEAD | kimi --quiet -p 'Summarize these changes'", workdir="~/project")
```

`--quiet` is a shortcut for `--print --output-format text --final-message-only`.

## Structured Output (stream-json)

```
# JSONL of assistant/tool messages
terminal(command="kimi --print --output-format=stream-json -p 'List all Python files and explain their role'", workdir="~/project")

# Bidirectional: read user messages from stdin continuously
terminal(command="my-tool | kimi --print --input-format=stream-json --output-format=stream-json", workdir="~/project")
```

### Exit codes (wire up retries in CI)

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Permanent error (config, auth, quota) — do **not** retry |
| `75` | Retryable error (429, 5xx, timeout) — back off and retry |

```
kimi --print -p "Run task"; code=$?
if [ $code -eq 75 ]; then sleep 10 && kimi --print -p "Run task"; fi
```

## Interactive Background Sessions

```
terminal(command="kimi", workdir="~/project", background=true, pty=true)
# session_id returned

process(action="submit", session_id="<id>", data="Refactor the auth module to use JWT")
process(action="poll",   session_id="<id>")
process(action="submit", session_id="<id>", data="Now add unit tests for the new JWT code")

# Clean exit: Ctrl+C or kill
process(action="write", session_id="<id>", data="\x03")
# or
process(action="kill",  session_id="<id>")
```

`/exit` is a valid slash command in Kimi (unlike opencode) but `Ctrl+C` / `process kill` is still the most reliable teardown.

## Session Management

| Flag | Purpose |
|------|---------|
| `--continue` / `-C` | Resume the most recent session in the current working dir |
| `--session <id>` / `--resume <id>` / `-S` / `-r` | Resume a specific session (or open picker without id) |
| `--session-id <uuid>` | Force a specific UUID for a new session |

Useful slash commands: `/new`, `/sessions` (alias `/resume`), `/fork`, `/undo`, `/title`, `/compact`, `/clear`, `/export`, `/import`.

```
# Archive a session as a ZIP
terminal(command="kimi export <session_id> -o /tmp/session.zip --yes")
```

## Plan Mode

Read-only exploration that writes an implementation plan to a plan file and waits for approval before touching code. The prompt shows a `📋` glyph and a blue `plan` badge.

```
terminal(command="kimi --print --plan -p 'Propose a migration from SQLAlchemy 1.x to 2.0'", workdir="~/project")
```

Slash: `/plan [on|off|view|clear]`. Config: `default_plan_mode = true` in `~/.kimi/config.toml` to start every new session in plan mode.

## Ralph Loop (Distinctive)

Kimi's autonomous iteration mode re-feeds the same task prompt until the agent prints a literal ` STOP ` marker or hits the iteration cap.

```
terminal(command="kimi --print -p 'Land the entire auth refactor: design, implement, test, commit' --max-ralph-iterations 20", workdir="/tmp/worktree-auth", background=true)
```

- `--max-ralph-iterations 0` disables Ralph Loop (default).
- `--max-ralph-iterations -1` is unlimited — always pair with a worktree and a provider-side cost cap.

## Agents & Skills

- `--agent default|okabe` — built-in agent presets (mutually exclusive with `--agent-file`).
- `--agent-file PATH` — custom agent definition.
- `--skills-dir PATH` (repeatable) — add extra skill trees on top of auto-discovered user/project skills.
- Slash: `/init` generates `AGENTS.md`; `/skill:<name>` loads a skill; `/flow:<name>` executes a flow skill.

```
terminal(command="kimi --print --agent okabe --skills-dir ./team-skills -p 'Review the PR on branch feat-x'", workdir="~/project")
```

## Approval & Safety

- `--yolo` / `-y` / `--yes` / `--auto-approve` — auto-approve every tool call. Implicit under `--print`.
- Runtime toggle: `/yolo` (yellow badge appears in status bar).
- Thinking mode: `--thinking` / `--no-thinking` (requires model support; inherits last session if unset).

## MCP Integration

```
# Load MCP servers for a single run (repeatable flags)
terminal(command="kimi --print --mcp-config-file ./mcp.json -p 'Query the prod database for stale users'", workdir="~/project")

# Manage global MCP config
terminal(command="kimi mcp add github -- npx @modelcontextprotocol/server-github")
```

Default config file: `~/.kimi/mcp.json`. Use `/mcp` in the TUI to inspect connected servers and exposed tools.

## Hooks & Plugins

- `/hooks` lists active hooks. Configure via `~/.kimi/config.toml` or project-level config.
- `kimi plugin` (Beta) manages plugins.

## Parallel Work Pattern

Use isolated worktrees and print-mode Kimi processes to fan out independent tasks:

```
terminal(command="git worktree add -b fix/issue-101 /tmp/issue-101 main", workdir="~/project")
terminal(command="git worktree add -b fix/issue-102 /tmp/issue-102 main", workdir="~/project")

terminal(command="kimi --print -y -p 'Fix issue #101 and commit' --max-ralph-iterations 10", workdir="/tmp/issue-101", background=true)
terminal(command="kimi --print -y -p 'Fix issue #102 and commit' --max-ralph-iterations 10", workdir="/tmp/issue-102", background=true)

process(action="list")
```

Never share one working directory across parallel Kimi sessions.

## PR Review Workflow

Kimi has no built-in `pr` subcommand (unlike opencode). Pipe the diff through print mode:

```
terminal(command="git diff origin/main...HEAD | kimi --quiet -p 'Review this diff. Report bugs, security issues, race conditions, and missing tests.'", workdir="~/repo")
```

For a fully isolated review:

```
REVIEW=$(mktemp -d) && git clone https://github.com/user/repo.git $REVIEW && cd $REVIEW && gh pr checkout 42 && git diff origin/main | kimi --quiet -p "Review PR #42 end-to-end"
```

## Subcommand Cheat Sheet

| Subcommand | Purpose |
|------------|---------|
| `kimi login` / `kimi logout` | Moonshot OAuth / sign-out |
| `kimi info` | Version + protocol info |
| `kimi acp` | Multi-session ACP server for IDE clients |
| `kimi web` | Browser UI server |
| `kimi term` | Toad terminal UI |
| `kimi mcp` | Manage MCP servers |
| `kimi plugin` | Plugin management (Beta) |
| `kimi export` | Archive a session as ZIP |
| `kimi vis` | Agent tracing visualizer (preview) |

## Other Useful Flags

`--model` / `-m`, `--work-dir` / `-w`, `--add-dir PATH` (repeatable; expands workspace scope for file tools), `--max-steps-per-turn`, `--max-retries-per-step`, `--verbose`, `--debug`, `--config` / `--config-file`, `--wire` (experimental Wire server mode).

## Pitfalls

- Interactive `kimi` (TUI) sessions require `pty=true`. `kimi --print` does not.
- Debug logging: `~/.kimi/logs/kimi.log` when using `--debug`
- `--print` implicitly enables `--yolo` — never point it at an untrusted or unversioned workdir.
- Exit code `75` means retry, not fail. Wire up retry loops in CI.
- `--max-ralph-iterations -1` can run for a very long time. Always pair with a worktree and a provider-side cost cap.
- Don't share one working directory across parallel Kimi sessions.
- These flag pairs are mutually exclusive: `--agent` / `--agent-file`, `--config` / `--config-file`, `--continue` / `--resume`.
- `/login` and `/model` only work with the default config file — they're disabled when you start with `--config` / `--config-file`.
- Large outputs: `--output-format stream-json` is friendlier than the full `json` blob for incremental parsing.

## Verification

Smoke test:

```
terminal(command="kimi --quiet -p 'Respond with exactly: KIMI_SMOKE_OK'")
```

Success criteria:

- Output includes `KIMI_SMOKE_OK`
- Command exits without auth/provider errors
- For code tasks: expected files changed and tests pass

## Rules

1. **Prefer `--print` or `--quiet` for one-shots** — no PTY, cleaner output; print mode implies YOLO auto-approve
2. **Use interactive background mode only when iteration is needed** — `background=true`, `pty=true`, then `process` poll/log/submit
3. **Always set `workdir`** — scope Kimi to the correct project root
4. **Isolate parallel runs** — git worktrees or separate dirs; never share one workdir across concurrent sessions
5. **Bound Ralph Loop** — treat `--max-ralph-iterations` as a long-running commitment; avoid unbounded `-1` without isolation
6. **Respect print-mode exit codes** — `75` = retry with back-off; `1` = permanent failure
7. **Monitor long tasks** — `process(action="poll"|"log")`; don't kill prematurely
8. **Exit interactive sessions cleanly** — `process(action="write", data="\x03")` or `process(action="kill")`
9. **Never point print/YOLO at untrusted trees** — use known repos or throwaway worktrees
10. **Report outcomes** — files changed, tests run, remaining risks
