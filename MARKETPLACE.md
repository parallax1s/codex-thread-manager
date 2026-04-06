# Marketplace Notes

This repository is prepared for two distribution paths:

## 1. Local Codex marketplace use

The repo includes a local marketplace manifest at:

- `.agents/plugins/marketplace.json`

That manifest makes the plugin discoverable as a repo-local plugin when the repository is opened in Codex and the plugin is enabled in `~/.codex/config.toml`.

This is the supported path that has been exercised during development.

## 2. Future official marketplace submission

At the time this repository was prepared, no public self-serve OpenAI documentation was found for publishing third-party Codex plugins into an official hosted marketplace.

The installed Codex app suggests two marketplace layers exist:

- local marketplaces controlled by the user or repo
- an OpenAI-curated marketplace synced by the app

This repository is therefore prepared for future submission, but not automatically publishable into an official marketplace from the outside today.

## Submission-ready assets already present

- plugin manifest: `plugins/codex-thread-manager/.codex-plugin/plugin.json`
- local marketplace entry: `.agents/plugins/marketplace.json`
- public source repo and license
- user-facing README
- tests covering shallow moves, deep moves, rollback, and safety gating

## Recommended next steps if an official intake path appears

1. Add screenshots or a short demo GIF.
2. Add a concise changelog or release notes.
3. Confirm any required branding or policy fields against the official intake requirements.
4. Submit the repository and manifest using the official process once it is documented.
