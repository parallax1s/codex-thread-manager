# Codex Thread Resumer Design

**Goal:** Build a local Codex plugin that can list desktop threads and force Codex.app to reopen a chosen thread by staging the `experimental_resume` startup setting.

**Why this shape:** The installed Codex app clearly supports internal thread resume, but the user-facing desktop UI does not expose a direct resume picker. The safest external leverage point we found is `experimental_resume` in `~/.codex/config.toml`, which points at an existing rollout JSONL file. Mutating storage tables or rollout logs would be brittle and unsafe; mutating the startup config is simpler and reversible.

## Architecture

The project will ship as a local plugin with an MCP server.

- The plugin manifest lives at `plugins/codex-thread-manager/.codex-plugin/plugin.json`.
- The MCP config lives at `plugins/codex-thread-manager/.mcp.json`.
- The MCP server is a small Python entrypoint that exposes tools for thread listing and resume staging.
- The server reads thread metadata from `~/.codex/state_5.sqlite`.
- The server stages resume targets by editing `~/.codex/config.toml`.
- The server can optionally restart `Codex.app` after staging the target.

## Tool Surface

V1 tools:

- `list_threads`: Return recent threads with id, title, cwd, archive flag, and rollout path.
- `get_thread`: Return one thread by id or exact title.
- `set_resume_target`: Write `experimental_resume` for a chosen thread without restarting the app.
- `clear_resume_target`: Remove the `experimental_resume` config entry.
- `resume_thread`: Set the resume target and optionally restart `Codex.app`.

## Safety Constraints

- Never modify thread rows in `state_5.sqlite`.
- Never rewrite rollout JSONL history.
- Always back up `config.toml` before changing it.
- Validate that the selected rollout path exists before writing config.
- Keep restart optional so staging can be tested separately.

## Packaging

The repo will use the plugin-friendly layout inferred from the installed Codex bundle:

- `plugins/codex-thread-manager/.codex-plugin/plugin.json`
- `plugins/codex-thread-manager/.mcp.json`
- `plugins/codex-thread-manager/server/`
- `plugins/codex-thread-manager/tests/`
- `.agents/plugins/marketplace.json`

This keeps the implementation testable locally and easy to publish later.

## Testing

Tests will cover:

- reading recent threads from a sqlite fixture
- resolving a thread by id or title
- staging and clearing `experimental_resume`
- backup creation for config edits
- restart command construction in dry-run mode

Live integration against the real `~/.codex` state stays manual and opt-in.
