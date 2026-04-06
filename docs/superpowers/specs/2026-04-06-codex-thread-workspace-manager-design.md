# Codex Thread Workspace Manager Design

**Goal:** Extend the local Codex plugin so it can re-home an existing desktop thread from one workspace to another, or create a workspace-based fork, while preserving actual thread usability across desktop hydration.

## Problem

The current plugin can stage a saved rollout for reopening through `experimental_resume`, but that does not reliably move the thread into a different desktop workspace view. A shallow metadata move in the `threads` table is also insufficient: the desktop app rehydrates the thread from the rollout file and snaps the thread back to its original workspace context.

In practice, the desired behavior is to make an existing thread show up under a new workspace root such as moving thread `019d53e5-9899-7b51-99cb-e6a39cf91ac2` from `Core` to `Helm` after the real repo context already changed, and to keep that thread usable after the user interacts with it.

The user needs two modes:

- **Move:** Permanently re-home a thread so the desktop app treats it as belonging to another workspace.
- **Fork:** Keep the original thread intact, but create a continuation in another workspace.

## Constraints

- Do not perform aggressive rollout rewriting.
- Make all metadata changes reversible.
- Let the user choose how much metadata to rewrite.
- Default behavior should fit the common case where the actual repo context already moved, and the app’s stored thread metadata is stale.

## Architecture

The plugin becomes a **thread workspace manager**. It still runs as a local plugin with one MCP server, but the server now operates on two coordinated data layers rather than relying on `experimental_resume` as the primary behavior.

### Data surfaces

- `~/.codex/state_5.sqlite`
  - `threads` table is the canonical source for thread workspace metadata.
- `~/.codex/sessions/.../rollout-*.jsonl`
  - the first `session_meta.payload.cwd` is part of the thread’s effective workspace identity during desktop hydration.
- `~/.codex/config.toml`
  - stays available for the old resume-staging path, but is no longer the main solution for workspace moves.

### Storage safety

Before mutating a thread row or rollout header, the plugin records a reversible backup of the original metadata in a local backup store. V1 can use a simple JSON backup file keyed by thread id and timestamp, plus a file backup for the rollout JSONL when a deep move is requested.

## Tool Surface

### 1. `inspect_thread_workspace`

Returns the current thread metadata relevant to workspace identity:

- `id`
- `title`
- `cwd`
- `rollout_path`
- `git_branch`
- `git_sha`
- `git_origin_url`
- `archived`
- `updated_at`

### 2. `preview_thread_move`

Shows the exact metadata diff that would be applied before making changes.

Inputs:

- `thread_id`
- `target_cwd`
- `mode`
  - `cwd_only`
  - `full_metadata`
- optional git overrides

Outputs:

- before/after metadata preview
- warnings about missing repo/git fields

### 3. `move_thread_to_workspace`

Permanently updates a thread to belong to a different workspace.

Inputs:

- `thread_id`
- `target_cwd`
- `mode`
  - `cwd_only`
  - `full_metadata`
- optional:
  - `git_branch`
  - `git_sha`
  - `git_origin_url`

Behavior:

- backup original row
- update `cwd`
- if `mode=full_metadata`, also update git metadata
- leave `rollout_path` unchanged
- if `deep_move=true`, patch the first rollout `session_meta.payload.cwd` to the new workspace path

### 4. `fork_thread_to_workspace`

Creates a workspace continuation rather than mutating the original thread.

Inputs:

- `thread_id`
- `target_cwd`
- optional git overrides

Behavior:

- create a new thread metadata record derived from the source
- assign a new thread id
- point to the same rollout path initially, or mark the fork as derived-from in backup metadata
- set the new `cwd` and chosen git metadata

Note:

This is more experimental than `move_thread_to_workspace` because the desktop app may assume stronger invariants around thread identity. V1 should implement it only if the state model is simple enough to do safely; otherwise the tool should return a clear `not yet supported safely` message.

### 5. `undo_last_thread_move`

Restores the last backup for a thread.

Inputs:

- `thread_id`

Behavior:

- restore the last saved metadata snapshot for that thread

## Metadata Modes

### `cwd_only`

Use when the user wants sidebar/workspace reassignment only and wants to preserve the original thread execution metadata.

Updates:

- `cwd`
- optionally the first rollout `session_meta.payload.cwd` when `deep_move=true`

Leaves unchanged:

- git metadata
- rollout path

### `full_metadata`

Use when the repo really moved and the app’s stored thread metadata is stale.

Updates:

- `cwd`
- `git_branch` if provided or detectable
- `git_sha` if provided or detectable
- `git_origin_url` if provided or detectable
- optionally the first rollout `session_meta.payload.cwd` when `deep_move=true`

Leaves unchanged:

- rollout path

This should be the default because it matches the user’s Helm case.

## Deep Move

### Why it is needed

Testing showed that updating only the `threads` row is not enough. The desktop app can initially group the thread under the new workspace, but once the thread is opened and fully hydrated, the rollout header reasserts the original workspace path and the thread snaps back.

### Deep move scope

For V1, deep move means:

- update the `threads` row
- patch only the first rollout line where `type == "session_meta"` and update `payload.cwd`

This is intentionally narrow. The plugin must not perform global search-and-replace over the rollout file in V1.

### Why not rewrite all path occurrences

Later rollout items may legitimately mention the old workspace path in tool calls, messages, or historical outputs. Rewriting those would risk corrupting history and making debugging harder. V1 only changes the minimum field that appears to control workspace hydration.

## Detection and Overrides

If the target workspace is a git repo, the plugin may detect:

- current branch
- HEAD sha
- origin URL

But the user can always override these values explicitly. This avoids hardcoding assumptions when the app failed to follow repo changes or when the repo is in a detached or dirty state.

## Backup and Undo

Backups are mandatory before mutation.

Each backup record should contain:

- thread id
- timestamp
- full original row fields that may be mutated
- operation type
- target cwd
- original rollout `session_meta.payload.cwd` when deep move is used
- path to the copied rollout backup file when deep move is used

Undo restores the most recent backup snapshot for the target thread.

## Error Handling

- If the thread does not exist: fail with a clear not-found error.
- If the target cwd does not exist: fail before any mutation.
- If git detection fails in `full_metadata` mode: continue with `cwd` update only if the user explicitly allows it, otherwise fail with a warning.
- If backup creation fails: abort the operation.
- If the rollout file cannot be parsed or its first line is not a `session_meta` object with `payload.cwd`: abort deep move and leave the thread row unchanged.

## Testing

Unit tests should cover:

- preview diff generation
- `cwd_only` move behavior
- `full_metadata` move behavior with explicit overrides
- backup creation
- undo restore
- deep move patching of the first rollout `session_meta.payload.cwd`
- refusal to perform aggressive whole-file rewriting

Integration checks should cover:

- moving a known thread row between two fake workspaces in a sqlite fixture
- verifying the row change is visible through the thread query layer
- patching a fixture rollout file and confirming only the first `session_meta.payload.cwd` changes

## Recommendation

Implement `move_thread_to_workspace`, `preview_thread_move`, `inspect_thread_workspace`, and `undo_last_thread_move` first. `move_thread_to_workspace` should support `deep_move=true` as the default path for real workspace re-homing. Keep `fork_thread_to_workspace` behind a safety gate until the thread identity invariants are better understood from the desktop app’s state model.
