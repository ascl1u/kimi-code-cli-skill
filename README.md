# kimi-code-cli — Agent Skill

A standalone skill that teaches agents to delegate coding tasks to [Kimi Code CLI](https://moonshotai.github.io/kimi-cli/en/guides/getting-started.html), Moonshot AI's terminal coding agent.

Use it for feature work, refactors, PR review, batch scripts, iterative multi-turn sessions, and long-running autonomous (Ralph Loop) runs — all orchestrated through Hermes's `terminal` and `process` tools.

## What this skill covers

- Print-mode (`kimi --print` / `--quiet`) one-shot automation
- Interactive background TUI sessions driven via the `process` tool
- `stream-json` structured I/O and CI-friendly exit codes (`0` / `1` / `75`)
- Session management (`--continue`, `--resume`, `kimi export`)
- Plan mode (`--plan`) and Ralph Loop (`--max-ralph-iterations`)
- Custom agents (`--agent`, `--agent-file`) and skill directories (`--skills-dir`)
- MCP integration (`--mcp-config-file`, `kimi mcp`)
- Parallel worktree fan-out and PR-review patterns

## Prerequisites

- Kimi Code CLI installed:

  ```
  curl -LsSf https://code.kimi.com/install.sh | bash
  # or, if you already have uv:
  uv tool install --python 3.13 kimi-cli
  ```

- Authenticated: `kimi login` (Moonshot OAuth) or `/login` on first TUI launch.
- Python 3.12–3.14 (3.13 recommended).

Verify: `kimi --version && kimi info`.

## License

MIT — see [LICENSE](./LICENSE).
