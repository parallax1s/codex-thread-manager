# Codex Thread Workspace Manager Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the local Codex plugin so it can move an existing desktop thread to a different workspace or create a workspace-based continuation, with reversible metadata changes and no rollout rewriting.

**Architecture:** Keep the plugin as one Python MCP server, but shift the core behavior from config staging to thread metadata management in `~/.codex/state_5.sqlite`. Add a backup store for reversible row mutations, implement preview and move first, and keep fork behavior behind a safety gate if database invariants turn out to be too weak for a safe first release.

**Tech Stack:** Python 3, sqlite3, JSON backup files, MCP stdio server, unittest

---

## Chunk 1: State Mutation Foundation

### Task 1: Add tests for workspace metadata moves

**Files:**
- Modify: `plugins/codex-thread-manager/tests/test_thread_store.py`
- Create: `plugins/codex-thread-manager/tests/test_backup_store.py`

- [ ] **Step 1: Write a failing test for previewing a `cwd_only` move**

Add a test that creates a thread fixture, asks for a preview into `/Users/mo/Desktop/Prj.nosync/Helm`, and expects the preview to show only `cwd` changing.

- [ ] **Step 2: Run the thread store tests to verify the new preview test fails**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_thread_store -v`
Expected: FAIL because preview support does not exist yet.

- [ ] **Step 3: Write a failing test for applying a `cwd_only` move**

Add a test that updates one fixture thread and expects the persisted row to have the new `cwd` with unchanged git metadata.

- [ ] **Step 4: Run the thread store tests to verify the move test fails**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_thread_store -v`
Expected: FAIL because move support does not exist yet.

- [ ] **Step 5: Write a failing test for applying a `full_metadata` move**

Add a test that passes explicit `git_branch`, `git_sha`, and `git_origin_url` values and expects those fields to be updated alongside `cwd`.

- [ ] **Step 6: Run the thread store tests to verify the full metadata test fails**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_thread_store -v`
Expected: FAIL because metadata mutation support does not exist yet.

- [ ] **Step 7: Write a failing test for backup creation**

Create a backup-store test expecting a backup snapshot to be written before mutation and retrievable for undo.

- [ ] **Step 8: Run the backup store tests to verify they fail**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_backup_store -v`
Expected: FAIL because backup store does not exist yet.

### Task 2: Implement the backup store

**Files:**
- Create: `plugins/codex-thread-manager/server/backup_store.py`
- Modify: `plugins/codex-thread-manager/server/models.py`
- Test: `plugins/codex-thread-manager/tests/test_backup_store.py`

- [ ] **Step 1: Implement a backup snapshot model**

Add a small dataclass for thread metadata backup records with thread id, timestamp, operation, target cwd, and original row data.

- [ ] **Step 2: Implement a JSON-backed backup store**

Write a minimal store that can append a snapshot and load the latest snapshot for a thread.

- [ ] **Step 3: Run the backup store tests to verify they pass**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_backup_store -v`
Expected: PASS.

## Chunk 2: Thread Mutation Logic

### Task 3: Extend the thread store for preview and move

**Files:**
- Modify: `plugins/codex-thread-manager/server/thread_store.py`
- Modify: `plugins/codex-thread-manager/server/models.py`
- Test: `plugins/codex-thread-manager/tests/test_thread_store.py`

- [ ] **Step 1: Implement a preview model for metadata diffs**

Add a data structure that returns `before`, `after`, and changed field names.

- [ ] **Step 2: Implement `preview_move`**

Support `cwd_only` and `full_metadata` modes using user-supplied overrides.

- [ ] **Step 3: Run the preview test to verify it passes**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_thread_store.ThreadStoreTests.test_previews_cwd_only_move -v`
Expected: PASS.

- [ ] **Step 4: Implement `move_thread`**

Update the `threads` row, backing up original values before mutation.

- [ ] **Step 5: Run the move tests to verify they pass**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_thread_store -v`
Expected: PASS for preview and move coverage.

- [ ] **Step 6: Implement undo restore**

Restore the latest backup snapshot for a thread.

- [ ] **Step 7: Add and run an undo test**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_thread_store.ThreadStoreTests.test_undo_restores_previous_thread_metadata -v`
Expected: PASS.

### Task 4: Gate fork behavior explicitly

**Files:**
- Modify: `plugins/codex-thread-manager/server/thread_store.py`
- Modify: `plugins/codex-thread-manager/tests/test_thread_store.py`

- [ ] **Step 1: Write a failing test for unsupported fork mode**

Add a test that asks to fork a thread and expects a specific safe “not supported yet” error.

- [ ] **Step 2: Run the fork test to verify it fails**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_thread_store.ThreadStoreTests.test_fork_is_rejected_until_supported_safely -v`
Expected: FAIL because the safe guardrail does not exist yet.

- [ ] **Step 3: Implement the safe rejection path**

Add a method or explicit exception describing that forking is withheld until invariants are understood.

- [ ] **Step 4: Run the fork test to verify it passes**

Run: `python3 -m unittest plugins.codex-thread-manager.tests.test_thread_store.ThreadStoreTests.test_fork_is_rejected_until_supported_safely -v`
Expected: PASS.

## Chunk 3: MCP Surface Update

### Task 5: Replace the old resume-centric tools with workspace tools

**Files:**
- Modify: `plugins/codex-thread-manager/server/main.py`
- Modify: `plugins/codex-thread-manager/.codex-plugin/plugin.json`
- Modify: `README.md`

- [ ] **Step 1: Write failing integration-style tests for the new tool names if needed**

If tool dispatch is simple enough, add a minimal test or direct invocation check for `inspect_thread_workspace`, `preview_thread_move`, `move_thread_to_workspace`, and `undo_last_thread_move`.

- [ ] **Step 2: Implement the new MCP tools**

Expose:
- `inspect_thread_workspace`
- `preview_thread_move`
- `move_thread_to_workspace`
- `undo_last_thread_move`
- `fork_thread_to_workspace` (safe rejection for now)

- [ ] **Step 3: Remove or clearly demote `experimental_resume` as the primary workflow**

Keep old config staging only if it remains useful as a secondary utility.

- [ ] **Step 4: Run the full unit suite**

Run: `python3 -m unittest discover -s plugins/codex-thread-manager/tests -v`
Expected: PASS with all tests green.

## Chunk 4: Manual Verification

### Task 6: Verify against the real Codex state

**Files:**
- Modify: none

- [ ] **Step 1: Run a non-destructive preview against the real thread**

Preview moving `019d53e5-9899-7b51-99cb-e6a39cf91ac2` into `/Users/mo/Desktop/Prj.nosync/Helm`.

- [ ] **Step 2: Apply a real move in `cwd_only` or `full_metadata` mode as requested**

Use the real thread id and record the before/after metadata.

- [ ] **Step 3: Query the thread row to verify the move landed**

Inspect the `threads` row in `~/.codex/state_5.sqlite`.

- [ ] **Step 4: Verify undo works**

Restore the previous metadata and confirm it returns to the original values.

- [ ] **Step 5: Reapply the chosen final move if the user wants it kept**

Leave the thread in its user-chosen final workspace state.
