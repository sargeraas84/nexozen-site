# macOS verification checklist

Everything in Nexozen's macOS path is implemented and syntax-checked, but **none of it
has been executed on real Apple hardware** — it was written on Windows, where macOS
packaging, `osascript`, and desktop-picture APIs cannot run. This checklist exists so a
single session on a Mac can confirm or falsify the whole feature set.

Work top to bottom. Items marked **blocking** must pass before macOS is described as
supported; the rest are quality gates.

---

## 0. Record the machine

| Field | Value |
| --- | --- |
| macOS version | <!-- e.g. 14.6.1 --> |
| Chip | <!-- Apple Silicon (M1/M2/M3/M4) or Intel --> |
| Model / year | |
| Display(s) | <!-- built-in only, or + external count --> |
| Nexozen version | `0.9.0` |
| Tester / date | |

---

## 1. Install and Gatekeeper (**blocking**)

Build: `Nexozen-0.9.0-mac-arm64.zip` (Apple Silicon) or `Nexozen-0.9.0-mac-x64.zip` (Intel).
Builds are **unsigned and un-notarized** by design at this stage.

1. Unzip, drag `Nexozen.app` into `/Applications`.
2. Double-click it. **Expected:** Gatekeeper refuses with *"Apple could not verify
   'Nexozen' is free of malware"* — this is the known, documented behavior, not a bug.
3. Right-click the app → **Open** → **Open** again. **Expected:** it launches, and
   subsequent double-clicks work.
4. If step 3 does not clear it, run `xattr -cr /Applications/Nexozen.app` in Terminal and
   retry. Confirm the quarantine flag was the cause:

   ```bash
   xattr -l /Applications/Nexozen.app   # expect com.apple.quarantine before clearing
   ```

5. Confirm the OS agrees it is unsigned (this is expected output, not a failure):

   ```bash
   spctl -a -vv /Applications/Nexozen.app
   # -> rejected (the code is valid but does not seem to be an app)
   # or "source=no usable signature"
   ```

- [ ] App launches from `/Applications` **blocking**
- [ ] No crash report generated in `~/Library/Logs/DiagnosticReports`
- [ ] App icon renders (not the generic Electron diamond) — Dock, ⌘-Tab, and About panel
- [ ] Menu bar item appears and its menu opens

> If the app bounces once in the Dock and dies, capture the reason before anything else:
> `"/Applications/Nexozen.app/Contents/MacOS/Nexozen"` in Terminal shows stderr.

---

## 2. Desktop layer (**blocking**)

The desktop layer is a frameless, non-activating, click-through window pinned behind the
desktop icons that renders the applied world live. macOS does not accept animated
wallpapers natively, so this layer *is* the live wallpaper on Mac.

1. Open **Settings → Desktop wallpaper** and confirm the host status reads as connected
   (the control host listens on `127.0.0.1:7477`).
2. Sanity-check the host directly:

   ```bash
   curl -s http://127.0.0.1:7477/status
   curl -s http://127.0.0.1:7477/current
   ```

3. Apply a shader world from Home, then apply an **AI-generated** wallpaper and confirm
   the desktop layer shows the image with **Ken Burns motion** (slow zoom/pan), not a
   static frame.
4. **Click-through:** click directly on a desktop icon and drag to select. The click must
   reach the desktop — the layer must never swallow it.
5. **Spaces and Mission Control:** open a second Desktop space, enter Mission Control,
   and trigger Exposé. The layer must stay behind icons and follow the active space.
6. **Fullscreen:** make any app fullscreen. The layer must not appear above it.
7. **Multiple displays:** with an external monitor attached, verify the layer covers the
   right screen and does not fight the menu bar (`NSWindow` level is near-desktop, below
   the menu bar).
8. **Pause/stop:** pause from the tray, confirm motion stops; stop, confirm the layer is
   gone and the normal macOS wallpaper is visible again.

- [ ] Host `/status` responds **blocking**
- [ ] Live shader world renders on the desktop **blocking**
- [ ] Generated image renders with Ken Burns motion **blocking**
- [ ] Click-through works (icons, selection, right-click)
- [ ] Behaves correctly across Spaces, Mission Control, fullscreen
- [ ] Correct display chosen when >1 monitor
- [ ] Pause and stop take effect immediately

---

## 3. Desktop-picture sync mode

macOS will not animate the system wallpaper, so this mode captures the live layer on a
timer (currently every **20 s**) and sets it as the actual system desktop picture via
`osascript`. It trades animation for native integration.

1. Enable **Sync to desktop picture** from the tray.
2. Wait one full cycle (~20 s + capture). Confirm the picture actually changed:

   ```bash
   osascript -e 'tell application "System Events" to get picture of current desktop'
   ```

3. Watch the change on screen — the desktop picture should swap to the new frame without
   a visible flash to black or a resize jump.
4. On **multiple displays**, record what happens. `osascript` sets one picture for all
   desktops; per-display pictures may be overwritten. *Expected risk: this is the most
   likely place to find a real defect.*
5. Disable the mode and confirm it stops writing files and **restores your previous
   wallpaper** (or report that you had to set it back manually).
6. Check where the frames land and clean up after testing:

   ```bash
   ls -la ~/Library/Application\ Support/Nexozen/ 2>/dev/null
   ls -la /tmp/nexozen* 2>/dev/null
   ```

- [ ] Picture changes on the timer
- [ ] Frame matches what the layer is rendering
- [ ] No flash / geometry jump on each swap
- [ ] Multiple displays: record actual behavior
- [ ] Disabling stops writes and restores the previous picture
- [ ] Frames are written outside tracked/synced folders (no iCloud sync churn)

---

## 4. Performance and battery

The desktop layer runs continuously, so cost matters more on a Mac than on the desktop.

1. With the layer running idle, read CPU/GPU in **Activity Monitor** (Energy tab).
2. Compare **FPS cap 60** vs a lower cap in **Settings → Performance**.
3. On a laptop, run 15 minutes on battery and record the drain delta with the layer on
   and off.
4. Note temperature/fan behavior — a wallpaper must never spin up fans at idle.

| Scenario | CPU % | Energy impact | Notes |
| --- | --- | --- | --- |
| Layer off, idle | | | |
| Shader world, 60 FPS cap | | | |
| Shader world, 30 FPS cap | | | |
| Picture sync mode on | | | |

- [ ] Idle CPU is low single digits with the layer on
- [ ] FPS cap visibly changes measured cost
- [ ] Battery drain increase is acceptable and stated in the README

---

## 5. App-wide behavior on macOS

- [ ] Library, Discover, AI Studio, Editor, Gadgets, Settings all render (no Windows-only assumptions in layout)
- [ ] **AI Studio** generates a real image and it appears in the library after a restart (disk persistence)
- [ ] Apply → Edit → Export loop works; exported WebM plays in QuickTime
- [ ] Tray menu items are the macOS wording (no "System tray" label leaks)
- [ ] Quitting from the tray exits cleanly and leaves no orphan process (`pgrep -fl Nexozen`)
- [ ] Reopening a second instance focuses the existing one instead of launching twice

---

## 6. Report template

Copy this into the issue/PR that closes the macOS verification, with raw output rather
than a summary of it.

```text
macOS:            <version> (<build>)
Chip:             <Apple Silicon model / Intel>
Nexozen:          0.9.0 (<zip name>)
Gatekeeper:       <passed after right-click Open | needed xattr -cr | failed: ...>
Control host:     <curl /api/status output>
Desktop layer:    <renders | fails: ...>
Ken Burns images: <motion confirmed | static | fails: ...>
Click-through:    <pass | fail: ...>
Spaces/fullscreen:<pass | fail: ...>
Picture sync:     <interval observed | restore behavior | multi-display behavior>
Perf (60 FPS):    <CPU % / energy impact>
Blockers:         <list, or "none">
```

---

## 7. Known-failing or unverified at the time of writing

State these as limitations rather than hiding them:

1. **Unsigned builds.** No Apple Developer certificate is wired up, so Gatekeeper
   friction is expected on every install until an identity is added.
2. **No animated system wallpaper.** macOS doesn't support it; that's why the desktop
   layer and picture-sync mode exist.
3. **Picture sync is a workaround.** Periodic capture is inherently less smooth than the
   layer, and it rewrites the user's desktop picture — it always restores or clearly
   reports what it changed.
4. **Distribution format.** macOS ships as ZIP today. A DMG is the conventional format
   and would be a small addition.
5. **Nothing above has run on hardware.** Every checkbox in this document starts
   unchecked for that reason.
