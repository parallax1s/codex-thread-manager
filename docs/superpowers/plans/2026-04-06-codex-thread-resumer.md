# Codex Thread Resumer Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local Codex plugin that can list desktop threads and reopen a selected thread in Codex.app by staging `experimental_resume`.

**Architecture:** Package the feature as a local plugin with one Python MCP server. The server reads thread metadata from Codex's sqlite state, writes a reversible `experimental_resume` config entry, and optionally restarts Codex.app. Avoid direct mutation of sqlite thread rows or rollout files.

**Tech Stack:** Python 3, sqlite3, TOML-safe text editing, Codex local plugin manifest, MCP stdio server

---

## Chunk 1: Project Scaffold

### Task 1: Create the plugin layout

**Files:**
- Create: `plugins/codex-thread-manager/.codex-plugin/plugin.json`
- Create: `plugins/codex-thread-manager/.mcp.json`
- Create: `plugins/codex-thread-manager/server/__init__.py`
- Create: `plugins/codex-thread-manager/server/models.py`
- Create: `plugins/codex-thread-manager/server/thread_store.py`
- Create: `plugins/codex-thread-manager/server/config_store.py`
- Create: `plugins/codex-thread-manager/server/app_control.py`
- Create: `plugins/codex-thread-manager/server/main.py`
- Create: `.agents/plugins/marketplace.json`

- [ ] Write the plugin manifest and MCP config with placeholder-safe but working local paths.
- [ ] Add a marketplace entry for local discovery.
- [ ] Keep metadata minimal and truthful for v1.

### Task 2: Add the test harness

**Files:**
- Create: `plugins/codex-thread-manager/tests/test_thread_store.py`
- Create: `plugins/codex-thread-manager/tests/test_config_store.py`
- Create: `plugins/codex-thread-manager/tests/test_app_control.py`

- [ ] Write failing tests for thread listing and lookup.
- [ ] Write failing tests for staging and clearing `experimental_resume`.
- [ ] Write failing tests for restart command generation in dry-run mode.

## Chunk 2: Core Behavior

### Task 3: Implement thread lookup

**Files:**
- Modify: `plugins/codex-thread-manager/server/models.py`
- Modify: `plugins/codex-thread-manager/server/thread_store.py`
- Test: `plugins/codex-thread-manager/tests/test_thread_store.py`

- [ ] Implement a typed thread record model.
- [ ] Implement listing recent threads from sqlite.
- [ ] Implement lookup by id and exact title.
- [ ] Run thread store tests until green.

### Task 4: Implement config mutation

**Files:**
- Modify: `plugins/codex-thread-manager/server/config_store.py`
- Test: `plugins/codex-thread-manager/tests/test_config_store.py`

- [ ] Implement backup creation for config edits.
- [ ] Implement set/replace behavior for `experimental_resume`.
- [ ] Implement clearing behavior.
- [ ] Run config store tests until green.

### Task 5: Implement restart support

**Files:**
- Modify: `plugins/codex-thread-manager/server/app_control.py`
- Test: `plugins/codex-thread-manager/tests/test_app_control.py`

- [ ] Implement restart command planning for `Codex.app`.
- [ ] Keep a dry-run mode for tests and safe inspection.
- [ ] Run restart tests until green.

## Chunk 3: MCP Surface

### Task 6: Implement MCP tools

**Files:**
- Modify: `plugins/codex-thread-manager/server/main.py`

- [ ] Expose `list_threads`.
- [ ] Expose `get_thread`.
- [ ] Expose `set_resume_target`.
- [ ] Expose `clear_resume_target`.
- [ ] Expose `resume_thread`.

### Task 7: Add usage docs

**Files:**
- Create: `README.md`

- [ ] Document layout, installation assumptions, and tool behavior.
- [ ] Document how to test with stage-only mode before restart mode.
- [ ] Document the open-source path and known limitations.

## Chunk 4: Verification

### Task 8: Run verification

**Files:**
- Modify: none

- [ ] Run the full test suite for the plugin package.
- [ ] Run one manual stage-only check against the real `~/.codex` state.
- [ ] Run one manual clear check.
- [ ] Summarize any app restart caveats in the README.
