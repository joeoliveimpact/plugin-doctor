# Changelog — plugin-doctor

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.2.0] — 2026-07-16

### Fixed
- **Stall A was clearing the wrong data store.** Field failure (07.16.26): the updater
  script cleared the Desktop display cache (IndexedDB), but the version that actually
  loads is pinned in `~/.claude/plugins/installed_plugins.json` (the plugin registry) —
  which the script never touched. Plugins stayed old while the fix reported success.

### Changed
- **Stall A rewritten around the two layers** (version registry vs. display cache).
  First-line fix is now the `claude` CLI: `claude plugin marketplace update` +
  `claude plugin update <plugin>@<marketplace>` — verified to unpin the registry
  instantly. The updater script (v1.1+, which now also runs the registry update when
  the CLI is present) and the uninstall → full-quit → reinstall UI path remain for
  Desktop-only users.

## [0.1.2] — 2026-07-07

### Changed
- **Now ships as its own standalone marketplace** at `joeoliveimpact/plugin-doctor` (moved out of `revxl-marketplace`). A repair tool for broken plugin installs shouldn't live inside the marketplace it repairs. Install with `claude plugin marketplace add joeoliveimpact/plugin-doctor` then `claude plugin install plugin-doctor@plugin-doctor`.
- **Genericized diagnosis.** Step 0 now asks which plugin and marketplace you're fixing instead of assuming `revxl-marketplace`; revxl remains as one example among the commands. Works for any plugin from any marketplace.

## [0.1.1] — 2026-07-03

### Changed
- **Stall A now fixes itself.** The doctor downloads the staged, fully reversible updater
  script from [Claude-marketplace-updater](https://github.com/joeoliveimpact/Claude-marketplace-updater)
  and launches it detached (`--stage1`, then `--stage2` if needed) — Claude Desktop quits,
  the cache clears, Desktop reopens, plugins re-sync. Works from a Claude Desktop chat or
  Claude Code; the user's session survives the restart. The previous CLI/UI walkthroughs
  remain as manual fallbacks when the script can't be fetched.

## [0.1.0] — 2026-06-30

### Added
- Initial release. One skill (`/plugin-doctor`) that diagnoses and fixes the two plugin-update stalls: **Stall A** — plugin stuck on an old version (marketplace refresh + force-reinstall, with CLI commands for Claude Code and step-by-step UI walkthrough for Claude Desktop, including the quit-and-reopen step the Desktop sync bug requires); **Stall B** — in-app directory install failing with `404 Not Found: plugin_<id>` (git-path install bypass + report path). Config-safety first: `${CLAUDE_PLUGIN_DATA}` and `~/.claude/revxl/` survive reinstalls, with an optional temp backup for belt-and-suspenders.
