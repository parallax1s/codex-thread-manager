# Codex Thread Workspace Manager

Local Codex plugin for inspecting and re-homing Codex desktop threads across workspaces.

Repository: [parallax1s/codex-thread-manager](https://github.com/parallax1s/codex-thread-manager)

## What it does

- lists recent local Codex desktop threads from `~/.codex/state_5.sqlite`
- inspects one thread's workspace-facing metadata
- previews a workspace move before mutating anything
- moves a thread to a new workspace with either `cwd_only` or `full_metadata` mode
- supports `deep_move` to patch the rollout header `session_meta.payload.cwd` so the app stops snapping the thread back
- stores reversible backups before each thread mutation
- restores the previous metadata through undo
- keeps the old `experimental_resume` helpers only as secondary utilities

## Layout

- `plugins/codex-thread-manager/.codex-plugin/plugin.json`: plugin manifest
- `plugins/codex-thread-manager/.mcp.json`: plugin MCP config
- `plugins/codex-thread-manager/server/`: Python MCP server
- `plugins/codex-thread-manager/tests/`: unit tests
- `.agents/plugins/marketplace.json`: local marketplace entry

## Install

Clone the repo and open it in Codex. The repo includes a local marketplace entry at `.agents/plugins/marketplace.json` and the plugin manifest at `plugins/codex-thread-manager/.codex-plugin/plugin.json`.

## Safety model

This project does mutate the `threads` sqlite table, but only the workspace-facing metadata fields needed for re-homing a thread. It does not rewrite rollout history. Before every thread move it writes a JSON backup snapshot, and undo restores the last saved metadata for that thread.

## Local testing

Run tests:

```bash
python3 -m unittest discover -s plugins/codex-thread-manager/tests -v
```

Workspace move workflow:

1. Use `preview_thread_move` with a thread id and target workspace path.
2. Use `move_thread_to_workspace` with `mode="cwd_only"` or `mode="full_metadata"`.
3. For real re-homing in the desktop app, set `deep_move=true`.
4. Inspect the thread row or the Codex app UI.
5. Use `undo_last_thread_move` if the re-home is wrong.

Legacy resume workflow:

1. Use `set_resume_target` or `resume_thread`.
2. The tool stages `experimental_resume`.
3. This path is retained only as a fallback utility and is not the main solution for desktop workspace moves.

## Notes

- `fork_thread_to_workspace` is intentionally safety-gated for now. The current implementation reports that forking is not yet supported safely.
- `deep_move` intentionally patches only the first rollout `session_meta.payload.cwd`. It does not rewrite historical tool output or other later path mentions.
