# `scripts/`

## `install-cli-agent.sh` / `uninstall-cli-agent.sh`

Run the standalone `screenpipe` CLI (`crates/screenpipe-engine`) as an
always-on macOS background service via `launchd`, independent of the Tauri
desktop app. Useful if you want `screenpipe record` running permanently
without the app UI, tray icon, or auto-updater.

```bash
./scripts/install-cli-agent.sh
./scripts/uninstall-cli-agent.sh
```

See the full usage, flags, and examples in the header comment of
`install-cli-agent.sh`.

### Why not the desktop app's "Auto-start" setting?

`tauri_plugin_autostart` (used by Settings → General → Auto-start) only sets
`RunAtLoad` — it starts the app once at login but does **not** relaunch it if
it crashes or is force-quit mid-session. `install-cli-agent.sh` installs a
`launchd` LaunchAgent with `KeepAlive`, so the process is relaunched on any
non-zero exit, not just at login.

### Why does it wrap the binary in a `.app` bundle?

System Settings → Privacy & Security's "+" button (Screen Recording /
Microphone / Accessibility) generally only accepts real `.app` bundles. A
bare Mach-O executable can be *selected* in the file picker but silently
fails to actually register in the permissions list. The script copies the
built binary into a minimal bundle at `~/Applications/ScreenpipeCLI.app` with
a proper `Info.plist` so it behaves like a normal app for TCC purposes.

### ⚠️ You need a real code-signing identity — ad-hoc signing is not enough

This is the part most likely to bite you, so it's called out here
explicitly, not just buried in a comment:

**Ad-hoc signing (`codesign -s -`, no Team ID) causes macOS TCC to
intermittently fail to re-validate the app on relaunch, silently dropping
previously-granted Screen Recording / Microphone / Accessibility access —
even when the binary and its `CDHash` haven't changed at all.**

This was confirmed hands-on while setting this up: an ad-hoc-signed build of
the wrapper bundle would get "ok" on Screen Recording / Microphone /
Accessibility on one run, then silently revert to "waiting for permission"
on a plain `launchctl kickstart` restart of the *exact same signed file* —
no rebuild, no re-signing, no file changes in between. Switching to a real
`codesign` identity (a free "Apple Development" cert, or a paid "Developer
ID Application" cert) fixed it completely across repeated restarts.

This mirrors a note already in this repo's `CLAUDE.md` ("macOS Dev Builds"):
the Tauri desktop app is deliberately signed with a developer certificate for
the same reason — "this ensures macOS TCC recognizes the app across
rebuilds."

`install-cli-agent.sh` auto-detects a usable identity via:
```bash
security find-identity -v -p codesigning
```
preferring `Developer ID Application`, falling back to `Apple Development`,
and finally falling back to ad-hoc signing with a loud warning if neither
exists. If you don't have either identity, get a free one via **Xcode →
Settings → Accounts → Manage Certificates → "+" → Apple Development**.

### Granting permissions

After installing, grant Screen Recording, Microphone, and Accessibility to
`~/Applications/ScreenpipeCLI.app` (the bundle, not the raw binary inside
it) via **System Settings → Privacy & Security → [category] → "+"**, using
**⌘⇧G** to paste the exact bundle path. Then restart the agent so the
already-running process picks up the newly granted permission:

```bash
launchctl kickstart -k gui/$(id -u)/pe.screenpi.cli
```

If a permission you already granted mysteriously stops working after a
restart, re-check `codesign -dv --verbose=4 ~/Applications/ScreenpipeCLI.app`
— if `Signature=adhoc` shows up instead of a `Developer ID Application` /
`Apple Development` authority chain, that's the ad-hoc flakiness described
above, not a bug in the script.

### Other gotchas handled by the script

- **PATH**: `launchd`'s default GUI-agent `PATH` is just
  `/usr/bin:/bin:/usr/sbin:/sbin` — no Homebrew. Without an explicit `PATH`
  in the plist, the binary can't find an already-installed `ffmpeg` at
  `/opt/homebrew/bin` and tries (and may fail) to auto-download its own copy.
  The generated plist sets `PATH` explicitly to include Homebrew's bin dirs.
- **Don't run the binary manually from a terminal first.** macOS attributes
  Screen Recording / Microphone / Accessibility prompts to the "responsible
  process" — for a terminal-launched process, that's the terminal app
  itself (iTerm, Zed, etc.), not the binary. Let `launchctl bootstrap` (via
  this script) be the first thing that ever runs it, so permissions get
  attributed to the app bundle instead.
