---
name: plugin-doctor:plugin-doctor
description: Diagnose and fix a Claude plugin that is stuck on an old version, won't update, or won't install from the in-app directory. Use when an installed plugin shows stale commands or old behavior after an update was published, when a plugin update appears stalled, or when installing from the directory fails with a "404 Not Found plugin_..." error. Trigger phrases include "my plugin won't update", "fix my plugin", "plugin is stuck", "I'm not seeing the new commands", "plugin install 404", "update my plugins".
---

# Plugin Doctor — fix stalled plugin updates

A plugin "stalls" three ways. Diagnose which one first; the fixes are completely different, and **one of them has no local fix at all** — recognising that early saves an hour of pointless clearing.

## Step 0 — Diagnose

**First, find out WHICH STORE the plugin loads from. There are three, and they are independent.**

| Store | Where | Who loads it |
|---|---|---|
| CLI registry | `~/.claude/plugins/installed_plugins.json` + `cache/` | Claude Code CLI, Desktop **Code tab** |
| **Cowork / agent-mode** | `%APPDATA%\Claude\local-agent-mode-sessions\<id>\rpm\plugin_<id>\` (macOS: `~/Library/Application Support/Claude/...`) | Desktop **Cowork tab** |
| Display cache | IndexedDB in the Claude app-data folder | the plugins panel only |

Check both real stores — **they disagree, and either one can be the stale one**:
- CLI: `claude plugin list`
- Cowork: read `rpm/manifest.json` for the plugin's name. If it appears there, Cowork has its **own** copy that the CLI cannot touch, and **the Cowork copy shadows the CLI one**.

Then ask:
1. **Which plugin**, and what marketplace it came from — ask, don't assume. `claude plugin list` shows each plugin's marketplace.
2. **Latest published version:** the plugin's CHANGELOG in its source repo, or the local catalog at `~/.claude/plugins/marketplaces/<marketplace>/.claude-plugin/marketplace.json`.

Then route:
- In the CLI store, installed < latest, updating does nothing → **Stall A** (registry pin).
- Fresh install from the in-app directory fails with `404 Not Found: plugin_<id>` → **Stall B** (hosted-record issue).
- Plugin is listed in Cowork's `rpm/manifest.json` and stays old no matter what you run locally → **Stall C** (server-side snapshot). Do not attempt local fixes; go straight to Stall C.

**Fast check for the most common Stall A signature** — the registry pointing at an old folder while the new one is already downloaded. Compare each entry's `installPath` in `installed_plugins.json` against the directories actually present in `cache/<marketplace>/<plugin>/`. A newer directory sitting there unused (especially one carrying an `.orphaned_at` marker) IS the diagnosis, in one read.

## Your config is safe (say this up front)

Plugin settings, story banks, and brand config live OUTSIDE the plugin's install folder — in `${CLAUDE_PLUGIN_DATA}` and (for REVXL engines) the shared `~/.claude/revxl/` folder. Uninstall/reinstall does not touch them. Belt-and-suspenders: before fixing, copy `~/.claude/revxl/` to a temp backup anyway; restore only if something looks off after.

## Stall A — stuck on an old version (two layers; know which one you're fixing)

A stuck update lives in one or both of two places:

- **Layer 1 — the version registry:** `~/.claude/plugins/installed_plugins.json`. This file decides which version actually LOADS. It is the usual culprit: the new version is often already downloaded into `~/.claude/plugins/cache/<marketplace>/<plugin>/` while the registry still points at the old folder.
- **Layer 2 — the Desktop display cache** (IndexedDB under the Claude app-data folder). This only affects what the plugins panel SHOWS. Clearing it does NOT unpin Layer 1 — a cache clear alone can report success while the plugin stays old (field-verified 07.16.26).

**First-line fix — the `claude` CLI (use whenever a shell is available, including a Claude Desktop chat that can run commands):**
1. Refresh the marketplace listing: `claude plugin marketplace update <marketplace-name>` (if that errors, `claude plugin marketplace remove <marketplace-name>` then re-add it). Example: `claude plugin marketplace update revxl-marketplace`.
2. Unpin the registry: `claude plugin update <plugin-name>@<marketplace-name>` — it reports `updated from X to Y`.
3. Verify: `claude plugin list` shows the new version. Restart Claude (or open a fresh session) to load it.

If `claude plugin update` says it's already on the latest but behavior is still old, the session is holding the old load — a fresh session/restart is the fix, not more clearing.

**Script fix — Claude Desktop without CLI access:**
A staged, fully reversible fixer lives at
https://github.com/joeoliveimpact/Claude-marketplace-updater (v1.1+ clears the display
cache AND runs the registry update automatically when the `claude` CLI is installed).
Before running it, tell the user what to expect: **Claude Desktop will quit and reopen
mid-fix** — their chat survives; they reopen it afterward and everything is right where
they left it. Then download and launch it **detached** (own process, so it survives the
restart):

- macOS:
  `curl -fsSL https://raw.githubusercontent.com/joeoliveimpact/Claude-marketplace-updater/main/fix-marketplace-sync-macos.command -o /tmp/cmu.command`
  then `(nohup bash /tmp/cmu.command --stage1 >/tmp/cmu-updater.log 2>&1 &)`
- Windows (PowerShell):
  `curl.exe -fsSL https://raw.githubusercontent.com/joeoliveimpact/Claude-marketplace-updater/main/fix-marketplace-sync-windows.bat -o "$env:TEMP\cmu.bat"`
  then `Start-Process "$env:TEMP\cmu.bat" -ArgumentList '--stage1'`

After Claude reopens: verify the plugin version (Settings → Plugins; macOS log at
`/tmp/cmu-updater.log`).

**Never escalate to `--stage2`.** It renames the entire Claude app-data folder
(`%APPDATA%\Claude` / `~/Library/Application Support/Claude`), which today also holds
`claude_desktop_config.json` (local MCP server config), the **Cowork plugin store**, and
the running Claude Code executable. On Windows it aborts on the open file handle with a
misleading "close Claude fully and re-run"; if it ever succeeded it would strand every
Desktop plugin and take local MCP config with it — the script's own promise that MCP
servers return after re-login is false for local config. Still old after stage 1? Go to
the uninstall/reinstall path below, or to Stall C if the plugin is Cowork-installed.

**No CLI at all (Desktop-only, script didn't stick):** uninstall/reinstall via the UI — this rewrites the registry entry, so it fixes Layer 1 without a terminal:
1. Open the plugins/extensions panel → find the plugin → uninstall it.
2. **Fully quit** and reopen Claude Desktop (the step people skip; a known Desktop sync bug makes updates stick until restart).
3. Reinstall the plugin from the directory.
4. Verify the version in the panel and check a NEW chat for the new commands.

## Stall B — in-app install 404 (`plugin_<id>` not found)

The repo is fine; the HOSTED directory record is orphaned. Two moves:
1. **Bypass now (Claude Code):** install by git path instead of the directory: `claude plugin install <plugin-name>@<marketplace-name>` after adding the marketplace by repo (`claude plugin marketplace add <org>/<repo>`). This works even when the in-app record 404s. Example: `claude plugin marketplace add joeoliveimpact/revxl-marketplace` then `claude plugin install <plugin-name>@revxl-marketplace`.
2. **Report it:** the marketplace owner needs to republish (a version-bump re-index) or escalate to Anthropic. Tell the user to report the exact `plugin_<id>` string and which plugin/version they tried.

Claude Desktop users who hit a 404 and can't use the CLI: report it (step 2) — the republish fix is on the publisher's side, usually same-day.

## Stall C — Cowork plugin frozen at an old version (no local fix exists)

**Symptom:** the plugin appears in Cowork's `rpm/manifest.json`, the Cowork tab keeps loading an old version, and nothing local moves it — not `claude plugin update`, not clearing caches, not deleting the `rpm` folder, not a full quit, not remove-and-re-add in the panel. The panel's Update button may be permanently greyed out.

**Cause (server-side, upstream bug — [#69683](https://github.com/anthropics/claude-code/issues/69683), open):** for marketplaces registered through Cowork, Anthropic's backend snapshots the GitHub repo at registration time and never re-pulls. The client is told "0 to download" and believes it is current. Removing and re-adding the marketplace is **deduplicated server-side** — the same marketplace id comes back, so the stale snapshot survives. Uninstall/reinstall rewrites files with fresh timestamps but the **same old content**.

Tell the user plainly: *"This one is on Anthropic's side. Nothing on your computer can fix it — the old copy is being served to you."* Do not burn their time on local clearing.

**The one remedy that works** ([#74609](https://github.com/anthropics/claude-code/issues/74609)): **remove the plugin from Cowork** (Customize → Skills → remove). With the Cowork copy gone, agent mode falls back to the CLI-installed copy, which you *can* update via Stall A. Re-adding it in Cowork re-enables the stale channel — so leave it removed.

**Diagnostic worth capturing** before removing: a manifest entry missing `updatedAtVerified` while its siblings have it is a strong marker of an orphaned snapshot. Note the `marketplaceName` too — if it names a marketplace whose catalog no longer lists that plugin, the snapshot predates the catalog change and is provably stale.

## After any fix

- Confirm the new version + commands in a FRESH session (old sessions keep the old plugin loaded).
- If a backup was taken and everything checks out, delete the temp backup.
- If the same plugin re-stalls on the next update, report that pattern — it's diagnostic gold for the publisher.
