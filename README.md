# Codex Thread Manager

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

### Install From Repo

1. Clone the repository:

```bash
git clone https://github.com/parallax1s/codex-thread-manager.git
cd codex-thread-manager
```

2. Open the repo in Codex Desktop.

3. Make sure the local marketplace entry exists at `.agents/plugins/marketplace.json`.

4. Enable the plugin in `~/.codex/config.toml` if Codex has not already added it:

```toml
[plugins."codex-thread-manager@local-codex-plugins"]
enabled = true
```

5. Restart Codex Desktop so it reloads the plugin list.

6. In a fresh thread, try prompts such as:

```text
Use codex-thread-manager to list my recent Codex desktop threads.
Use codex-thread-manager to preview moving thread <id> to /path/to/workspace with full_metadata mode and deep_move true.
```

### Local Marketplace Entry

This repo ships with a local marketplace manifest at `.agents/plugins/marketplace.json`:

```json
{
  "name": "local-codex-plugins",
  "plugins": [
    {
      "name": "codex-thread-manager",
      "source": {
        "source": "local",
        "path": "./plugins/codex-thread-manager"
      }
    }
  ]
}
```

Codex uses that manifest as a repo-local marketplace, not as a public OpenAI-hosted listing.

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
- There is no documented self-serve OpenAI marketplace publishing flow at the time this repo was prepared. This project is ready for local marketplace use and public source distribution.
