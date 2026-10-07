# Handoff: io.github.rk4500.archer-bar

## What this is

A clone of the built-in `omarchy.bar` (made via `omarchy plugin clone omarchy.bar`,
see `PATCHES.md` for the required-property fix that was needed just to get the
clone loading at all). This is the plugin actually driving the desktop bar —
confirmed live via `quickshell ipc show` (a temporary unique probe function
added to the IPC handler, restarted, and checked for in the live target list).

It is **not** a git-managed plugin in Omarchy's sense (that term means
installed via `omarchy plugin add <git-url>`, which tracks a remote and
supports `omarchy plugin update <id>`). A `clone` is a one-time file copy with
no `.git` and no update path — `omarchy plugin update io.github.rk4500.archer-bar` will refuse
outright ("not a git checkout"). This repo was `git init`'d locally after the
fact, purely for the user's own change history; it has no relationship to
Omarchy's own update machinery and Omarchy will never fetch into or touch it.

## Staying stable as the built-in bar changes upstream

Because this is a frozen copy, nothing here changes out from under you on a
system/package update — that's the good news. The tradeoff: it also never
picks up upstream bug fixes or new features on its own. Two risks to watch:

1. **Silent drift**: the built-in bar improves over time; this fork doesn't.
   Periodically diff against the current upstream source to see what moved:

   ```
   diff -u /usr/share/omarchy/shell/plugins/bar/Bar.qml \
           ~/.config/omarchy/plugins/io.github.rk4500.archer-bar/Bar.qml
   ```

   (Source path confirmed via `omarchy-plugin-catalog | jq '.[] | select(.id=="omarchy.bar")'`.)

2. **Host interface changes**: `shell.qml`'s loader contract (what properties
   it assigns, when, and how) could change in a future Omarchy release the
   same way it apparently did before — see `PATCHES.md`'s required-property
   story, which is exactly this failure mode. If the bar silently stops
   loading after a system update, that interface mismatch is the first thing
   to check, same diagnostic approach as the original PATCHES.md fix.

There's no automatic guard against either. The git history below is the
practical mitigation: a known-good, bisectable trail so a future problem can
be isolated to a specific change instead of debugged from scratch.

## What changed this session (2026-10-06)

All in `Bar.qml`, confirmed live only after discovering `io.github.rk4500.archer-bar` wasn't
actually the active bar for most of the session (see git log for the
individual commits and reasoning):

- **Auto transparency by window count**: bar goes opaque only when the
  focused workspace has exactly one non-floating window, clear otherwise.
  Driven by `Hyprland.focusedWorkspace`/`toplevels`, refreshed on relevant
  `Hyprland.onRawEvent` names (`openwindow`, `closewindow`, `movewindow`,
  `changefloatingmode`, `workspace`, `focusedmon`), same debounce-then-bump
  pattern as the `tornikegomareli.spaces` plugin's `revision` mechanism.
- **Fixed the transparency toggle snapping instead of fading**: the
  `PanelWindow`'s `color:` binding (`root.transparent ? "transparent" :
  root.background`) had no `Behavior` on it at all — the pre-existing
  `Behavior on background` was on a *different*, mostly-static property, so
  it never animated anything that mattered. Added `Behavior on color` on the
  actual `PanelWindow`, currently 200ms `InOutCubic` (280ms -> 150ms -> 200ms
  across a couple of tuning requests — feel free to keep adjusting via that
  one line).
- **Fixed ~0.5s of dead air before the fade even started**: `transparent`
  used to flip only after `omarchy-bar-text-color` (an external process doing
  pixel sampling for legible text color, measured ~0.5s wall-clock) finished.
  That process has nothing to do with whether the background should be
  visible — it only decides the text color's final shade. `transparent` now
  flips immediately in `setRequestedTransparency`; the sampled text color
  still arrives ~0.5s later and fades in on its own (`barForeground` already
  had a working `Behavior`), independent of the background now.

## Open/next (discussed, not built)

- **Color-matched bar background**: sample the screen region behind the bar
  (same `grim`-based approach `omarchy-bar-text-color` already uses) and set
  `root.background` to the dominant/average color when the single-tiled-window
  rule holds, so a maximized app's own chrome color bleeds into the bar.
  Feasible, reuses existing plumbing. Caveat: that sampling process is the
  same ~0.5s op already fought once this session — fine on focus-change/
  resize events with debouncing, not something to run continuously.
- ~~Hover tooltips for wifi/bluetooth/battery~~ — **done**. Turned out to
  need even less than expected: these three widgets' bar icon is a
  `BarIconButton` (extends the shared `WidgetButton`), which already wires
  `onEntered -> root.bar.showTooltip(root, root.tooltipText)` generically —
  nothing to build, just set `tooltipText:` per widget. Cloned each into its
  own `archer.*` fork (same pattern as this one — `omarchy plugin clone
  omarchy.network` / `omarchy.bluetooth` / `omarchy.power`, each now its own
  git repo in `~/.config/omarchy/plugins/`) since they're system files under
  `/usr/share/omarchy/shell/plugins/panels/` and `.../services/battery/`,
  not something to hand-patch in place. Network shows SSID + signal%
  (or interface name for ethernet); bluetooth shows connected device name(s)
  via the existing `Model.deviceLabel()` helper; power shows percentage +
  `root.modeLabel()`. `omarchy plugin clone` swapped each one's `shell.json`
  layout entry in place automatically (unlike the bar, where `.bar.id`
  needed `omarchy bar use` by hand) — confirmed via a clean `grim` screenshot
  of the bar's right section post-restart, all five icons intact.

## Git workflow going forward

Plain local repo, no remote. Commit discrete, explained changes as you go —
that's the whole point of having this tracked. Before any future bulk
re-clone or manual upstream merge, `git stash`/commit first so nothing here
is ever at risk of being silently overwritten by a copy operation.
