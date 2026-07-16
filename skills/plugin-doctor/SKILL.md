---
name: plugin-doctor:plugin-doctor
description: Diagnose and fix a Claude plugin that is stuck on an old version, won't update, or won't install from the in-app directory. Use when an installed plugin shows stale commands or old behavior after an update was published, when a plugin update appears stalled, or when installing from the directory fails with a "404 Not Found plugin_..." error. Trigger phrases include "my plugin won't update", "fix my plugin", "plugin is stuck", "I'm not seeing the new commands", "plugin install 404", "update my plugins".
---

# Plugin Doctor — fix stalled plugin updates

A plugin "stalls" two ways. Diagnose which one first; the fixes are different.

## Step 0 — Diagnose

Ask (or check) three things:
1. **Which plugin**, and what marketplace it came from — ask the user for both, don't assume. The marketplace is the `@<marketplace>` suffix they used at install (e.g. `plugin-doctor`, `revxl-marketplace`, or whatever they added); if they don't know it, `claude plugin list` shows each plugin's marketplace.
2. **Installed version:** Claude Code → run `claude plugin list` and read the version. Claude Desktop → open the plugins/extensions panel and read the version shown.
3. **Latest published version:** check the marketplace catalog — the plugin's CHANGELOG in its source repo. (Example: a revxl-marketplace plugin's CHANGELOG lives at https://github.com/joeoliveimpact/revxl-marketplace/tree/main/plugins/.)

Then route:
- Installed < latest, and updating does nothing → **Stall A** (stuck update).
- Fresh install from the in-app directory fails with `404 Not Found: plugin_<id>` → **Stall B** (hosted-directory record issue).

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
`/tmp/cmu-updater.log`). Still old? Re-run with `--stage2` (full local reset — bigger
clear, still reversible, user re-logs-in).

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

## After any fix

- Confirm the new version + commands in a FRESH session (old sessions keep the old plugin loaded).
- If a backup was taken and everything checks out, delete the temp backup.
- If the same plugin re-stalls on the next update, report that pattern — it's diagnostic gold for the publisher.
