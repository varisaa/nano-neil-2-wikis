# USB camera: dedicated Openbox desktop with a profile-picker menu

**Status: implemented, live-tested (2026-08-28).**

## Deviations found during implementation

**1. Camera window crashed/never appeared (blank screen) on first boot.**
OpenCV's GTK highgui backend tries to activate an AT-SPI accessibility
D-Bus session the first time it creates a window. In this bare Xvfb +
Openbox environment there's no pre-existing desktop D-Bus session to reuse
(unlike an interactive SSH shell, which inherits one), so a fresh one gets
spun up via `dbus-launch`/`at-spi2-registryd` — and that stalled/silently
killed the camera process (visible as a ~1m45s gap in the journal between
the process starting and it actually opening the pipeline, then it vanished
with no error). Fixed with the standard `export NO_AT_BRIDGE=1` workaround,
added to `/home/neil/bin/camera-launch.sh` before the `exec python3`.

**2. Openbox menu was unreachable via right-click.** The camera window
(1920x1080, since `simple_camera.py`'s YUYV/MJPG pipelines don't downscale
before `imshow`) is bigger than the 390x844 canvas and covers it entirely —
there's no exposed root/desktop background left to right-click on, which is
what the default Root-context menu binding requires. Fixed by adding a
`ShowMenu` action to the `Client` context's right-click bind in
`/home/neil/.config/openbox-camera/rc.xml`, so right-clicking directly on
the camera window (not just empty desktop) also opens the root menu.

Both confirmed live: window stayed up and painted frames, menu opens via
right-click on the video itself, and switching between at least three
profiles in a row worked cleanly (old process stopped, new one started, no
"device busy" errors).

**3. Camera window and `xterm` both overflowed the phone's screen
boundary in portrait mode.** `simple_camera.py`'s USB pipelines (`cv2`
`WINDOW_AUTOSIZE`) sized the window to the full capture resolution
(1920x1080), and `xterm`'s default geometry (80x24) is also wider than
390px — both badly overflowed the 390x844 canvas. Fixed two ways:
- Added `--display-width`/`--display-height` options to
  `simple_camera.py` (a `videoscale` stage in the USB gstreamer pipelines,
  only when both are given — no change to existing callers/behavior).
  `camera-launch.sh` now always appends `--display-width 380
  --display-height 800` (a small margin inside the 390x844 canvas) since
  it's only ever used for this dedicated desktop. Confirmed via `xwininfo`:
  window is now exactly `380x800`, comfortably inside the canvas with the
  titlebar fully visible.
- `menu.xml`'s Terminal entry now launches `xterm -geometry 36x40+0+20`
  instead of bare `xterm`. Confirmed: window is `220x524`, well inside the
  canvas.

**Reboot-survival confirmed as a side effect**: the machine rebooted
mid-session (unplanned, ~22:49) and all three systemd units
(`xvfb-camera-portrait`/`landscape`, `desktop-camera`, `x11vnc-camera`)
came back up automatically with no console login needed, satisfying
verification step 8 below. The USB camera itself took a few reconnect
cycles after boot before `uvcvideo` settled and `/dev/v4l/by-id/...`
reappeared (worth keeping in mind if the camera menu items briefly fail
with "Unable to open camera" right after a reboot - retry from the menu
once `lsusb`/`/dev/v4l/by-id/` show it back).

**4. Video was stretched/distorted in portrait mode.** `--display-width
380 --display-height 800` forced the 16:9 capture into an exact portrait
box via `videoscale`, squashing the aspect ratio. Fixed by adding a
`fit_within()` helper to `simple_camera.py` that computes a
letterboxed size preserving the source aspect ratio (like CSS
`object-fit: contain`) instead of stretching to the exact box. Confirmed:
window is now `380x212` in portrait (correct 16:9), not distorted.

**5. No landscape support - rotating the phone only used half the
screen.** The canvas was single-orientation (390x844 only), so a
landscape-rotated VNC client just showed that narrow portrait framebuffer
inside a wider window. Added full portrait/landscape support, mirroring
`headless-vnc-ipad-iphone-setup.md`'s proven pattern:
- Split into `xvfb-camera-portrait.service` (390x844) and
  `xvfb-camera-landscape.service` (844x390), each with
  `Conflicts=` on the other so starting one stops the other. Portrait is
  the enabled/boot default; landscape starts disabled (same convention as
  the sibling machine).
- Root-owned `/home/neil/bin/set-camera-orientation.sh portrait|landscape`
  starts the target unit and persists the choice via
  `enable`/`disable`, so the last-picked orientation survives a reboot.
  A scoped NOPASSWD sudoers rule (`/etc/sudoers.d/camera-orientation`,
  same pattern as the sibling's `/etc/sudoers.d/vnc-orientation`) lets
  `neil` run only those two exact invocations.
- `desktop-camera.service`/`x11vnc-camera.service` changed from
  `BindsTo=`+`on-failure` to ordering-only `After=` + `Restart=always` -
  avoids the exact `Wants=`-fighting-`Conflicts=` bug the sibling
  machine's doc already found and fixed; openbox/x11vnc now just
  auto-reconnect within a couple seconds whenever the active Xvfb swaps.
- `menu.xml` gained "Orientation: Portrait"/"Orientation: Landscape"
  items (`sudo set-camera-orientation.sh ...`).
- `camera-launch.sh` no longer hardcodes portrait's 380x800 - it queries
  the *current* screen size via `xdotool getdisplaygeometry` at launch
  time and fits to that (minus a small margin), so profile-switching and
  orientation-switching both always produce a correctly-sized,
  correctly-proportioned window.

Confirmed live end-to-end: switched landscape -> portrait -> landscape
several times via both direct script invocation and the actual menu path
(keyboard-nav-select, to sidestep `scrot` stealing focus mid-test);
`xrandr`, the camera window geometry, and port 5901 all tracked correctly
through every switch with no manual intervention.

**6. Added recording, via a new "Toggle Recording" menu item.**
`simple_camera.py`'s `show_camera()` now has a `SIGUSR1` handler that
flips a recording flag; the main loop opens/closes a `cv2.VideoWriter`
(`mp4v`, ~20fps - an approximation of achieved throughput, not the
pipeline's nominal framerate) to a timestamped file in
`~/work/camera/recordings/` when that flag changes, and draws a red "REC"
dot/label on the *displayed* frame only (not the recorded one) so it's
visible in the VNC view. Pressing 'r' does the same thing for interactive/
keyboard use. `camera-launch.sh` now writes its pid to
`/tmp/simple-camera.pid` (before `exec`, so the pid carries over) for the
menu item to signal via `kill -USR1 $(cat /tmp/simple-camera.pid)`.

Also added a `SIGTERM` handler that raises `SystemExit` instead of letting
Python's default (immediate kill, skipping `finally`) run - otherwise
`camera-launch.sh`'s `pkill` on every profile/orientation switch would
leave an in-progress recording's `.mp4` unfinalized. Confirmed: toggled
recording on/off/on across a profile switch via both the menu path and a
direct signal, all `.mp4` files came back valid and playable (`cv2`
reported correct frame counts and dimensions matching the window size).

**7. Playback: "Play Recording..." submenu + a missing codec.**
Added `/home/neil/bin/list-recordings-menu.sh`, an Openbox "pipe menu"
(`<menu execute="...">` in `menu.xml`) that lists
`~/work/camera/recordings/*.mp4` newest-first, regenerated fresh every
time the submenu opens, each item launching that file in Totem
(`NO_AT_BRIDGE=1`, same reasoning as `camera-launch.sh`). Totem itself was
already installed, but opening a recording first failed with "Unable to
play the file - MPEG-4 Video (Simple Profile) decoder is required" -
`gstreamer1.0-libav` (FFmpeg-based decoders) wasn't installed, and nothing
else on this system decodes the `mp4v` fourcc `cv2.VideoWriter` writes.
Installed `gstreamer1.0-libav`; confirmed a recording plays back correctly
afterward.

**8. Terminal geometry was fixed, not orientation-aware.** The Terminal
menu item's `xterm -geometry 36x40+0+20` was sized for portrait only
(fine at 390x844, badly undersized/wrong after switching to landscape
844x390, where it left most of the canvas unused instead of overflowing).
Added `/home/neil/bin/open-camera-terminal.sh`, which queries the current
screen size the same way `camera-launch.sh` does and computes columns/rows
from it (~6x13px per xterm character cell), so the terminal now properly
fills whichever orientation is active. Confirmed: `382x797` in portrait,
`838x342` in landscape - both comfortably filling their canvas.

**Follow-up**: filling the canvas that closely read as "full screen" -
right-click-on-the-terminal-itself to reach the menu still worked fine
(confirmed via screenshot), but per user preference
`open-camera-terminal.sh` now targets ~75% of the canvas instead, centered
(`offset = (screen - content) / 2`), leaving visible desktop margin on all
sides. Confirmed: `292x628` centered in portrait, `634x290` centered in
landscape.

**9. "Terminal: Resume Claude" menu item.** Added
`/home/neil/bin/open-claude-resume-terminal.sh` (same sizing/centering as
`open-camera-terminal.sh`, `cd ~/work` then `exec xterm ... -e claude
--resume`) so a Claude Code session can be picked up directly from this
VNC menu. Deliberately uses `--resume` (interactive picker) rather than
`--continue` (auto-attaches to the most recent session) - if that most
recent session is this very one and it's still running elsewhere, two
`claude` processes writing the same session transcript concurrently is
asking for trouble; the picker lets the choice be made knowingly instead.
Confirmed live: opens correctly, session picker renders and lists this
conversation.

**Follow-up fix**: first real use hit `xterm: Can't execvp claude` -
`xterm -e` doesn't source `.bashrc`/`.profile`, so `~/.local/bin` (where
`claude` actually lives - confirmed via `which claude`) isn't on `PATH` in
that bare exec context. Fixed by calling the full path
(`/home/neil/.local/bin/claude`) instead of the bare command name.
Confirmed working afterward.

**Follow-up refinement**: user wanted the menu item to jump directly into
*this* session, not the picker. Found this session already has a custom
title ("Camera-2 cameraman", stored in
`~/.claude/projects/-home-neil-work/<session-id>/custom-title.json`) -
that's where "camera-2" came from. `claude --resume <name>` only
pre-filters the interactive picker though, per `claude --help` ("Resume a
conversation by session ID, **or** open interactive picker with optional
search term") - it doesn't guarantee skipping straight to a single match.
The only value that truly skips the picker is the exact session ID, so
`open-claude-resume-terminal.sh` now runs `claude --resume
ae7e5994-b72c-4cef-be65-22f44f649487` (this session's actual ID) instead
of a name/search term. Menu label updated to "Resume: Camera-2 cameraman"
for clarity. **Not live-tested** (deliberately) - actually running this
would open a second `claude` process attached to this exact, currently-
open session, risking concurrent writes to the same transcript; confirmed
the flag syntax via `--help` instead. Worth a real test next time this
session isn't also open elsewhere.

**Follow-up**: xterm's default background (light/white on this system)
was washing out some text. Added `-bg black -fg white` to both
`open-camera-terminal.sh` and `open-claude-resume-terminal.sh`'s `xterm`
invocations. Confirmed live - clean black background, white text.

**Follow-up**: the ~75%-height terminal extended too far down, getting
covered by a phone's on-screen keyboard when tapping in to type. Reduced
to ~45% height and anchored near the top (`offset_y=40`, no longer
vertically centered) in both scripts, leaving the lower half of the
canvas free. Confirmed live: `292x381` in portrait (was `292x628`),
positioned near the top with open space below. This sizing/positioning
was kept even after the onboard reversal below - still reasonable
regardless of which keyboard shows.

**Tried and reverted: onboard (in-session on-screen keyboard).** When the
native iPhone keyboard didn't reliably appear, tried installing `onboard`
(already present) to run *inside* the X session instead, auto-launched by
both terminal scripts and toggleable from the menu - confirmed live that
it rendered correctly (full-width across the bottom, right below the
terminal) and that taps on it correctly delivered keystrokes to the
terminal. But this broke the native iPhone keyboard integration that had
been working before onboard was added (exact mechanism unclear - VNC
clients' native-keyboard heuristics are opaque from the server side).
Per user preference, fully reverted: removed `onboard`'s auto-launch from
both `open-camera-terminal.sh`/`open-claude-resume-terminal.sh`, deleted
the now-unused `toggle-keyboard.sh` and its menu item, and killed the
running `onboard` process. Back to relying on the phone's native
keyboard - if it still doesn't show reliably, that's client-side (check
the VNC app's own keyboard toggle button) rather than something to chase
on this end again.

**The revert alone didn't stick - onboard kept coming back.** Root
cause: `/etc/xdg/autostart/onboard-autostart.desktop` (system-wide,
`X-GNOME-AutoRestart=true`), which restarted it every time it was killed
- independent of removing the explicit `onboard &` calls from the two
terminal scripts. Its own conditions (`OnlyShowIn=Unity;MATE`, and
`AutostartCondition=GSettings
org.gnome.desktop.a11y.applications screen-keyboard-enabled`, confirmed
`false`) should have prevented it from running in this bare Xvfb+Openbox
session at all, but something in the GNOME-portal stack this session
pulls in (`xdg-desktop-portal-gnome` et al.) isn't respecting that.
Fixed with a user-level override, `~/.config/autostart/
onboard-autostart.desktop` (`Hidden=true`) - the standard XDG mechanism
to disable one specific autostart entry without touching the system-wide
file (which the real `:0` GNOME desktop still reads, in case the a11y
screen-keyboard setting is ever legitimately turned on there). Confirmed
killed and not respawning after 15s, and still gone after a full
`desktop-camera.service` restart.

**Re-added as an on-demand toggle only.** The native iPhone keyboard
turned out not to be reliably working regardless of onboard, so there was
no longer a real tradeoff. Recreated `toggle-keyboard.sh` and its
"Toggle On-Screen Keyboard" menu item (same as before), but deliberately
did **not** re-add the auto-launch calls to `open-camera-terminal.sh`/
`open-claude-resume-terminal.sh` - onboard now only appears when
explicitly toggled on, never automatically. The `/etc/xdg/autostart`
override from the previous fix stays in place, so it still won't
self-launch on its own. Confirmed live: toggle correctly shows then hides
it, and it stays off by default after an openbox reconfigure.

**Open item, not yet root-caused**: sometime during this Totem/playback
testing, the `simple_camera.py` process disappeared with no trace in
`journalctl -u desktop-camera.service` (no traceback, no "X connection
broken", no `SIGTERM` handler message) and no OOM-kill in `dmesg` or the
system journal either - just gone. Memory was tight at the time (`free -h`
showed only ~250MB free, ~1.7GB available) while Totem pulled in Tracker/
GVFS volume-monitor services, so memory pressure is a plausible suspect
even without a logged OOM event, but this isn't confirmed. `desktop-camera`
itself didn't restart (only its child died), so nothing auto-recovered it -
had to `systemctl restart desktop-camera.service` manually. If this
recurs, worth checking `dmesg`/`journalctl -k` right after and considering
whether Totem should be closed before switching camera profiles, or
whether `simple_camera.py` itself needs its own supervision separate from
openbox's one-shot `--startup`.

## Context

The user has been running `CSI-Camera/simple_camera.py` interactively
against the USB Global Shutter Camera on `nano-neil-2`, with several saved
`.gpfl` control profiles under `camera/camera-settings/` (`defaults`,
`night-defaults`, `zero-backlight`, `sunglight`, `backlight`, `manual-exp`)
for different lighting conditions. The ask evolved over this planning
session:

1. Make the camera viewer launch automatically instead of needing an SSH
   session to run it by hand each time.
2. Be viewable remotely from an iPad/iPhone via a VNC-style app (RealVNC
   Viewer / Screens) — see `wikis/plans/vnc-screen-sharing.md` for the
   general-purpose GNOME desktop sharing already enabled on this machine.
   That mirrors the *whole* monitor session, which turned out to be more
   than wanted for just watching the camera.
3. Refined ask: a **separate, simplified desktop** just for the camera —
   its own lightweight Openbox session, its own VNC port, not the full
   GNOME session.
4. Sized like the sibling machine's iPhone 14 virtual desktop (390x844) —
   see `wikis/plans/headless-vnc-ipad-iphone-setup.md` for that precedent.
5. Final refinement: instead of one fixed autostarted camera command, expose
   an **Openbox right-click menu** to launch/relaunch the camera with a
   chosen `.gpfl` profile — since the camera is USB (only one process can
   hold the device at a time), switching profiles means stopping whichever
   instance is running and starting the new one.

This reuses the same building blocks as `headless-vnc-ipad-iphone-setup.md`
(Xvfb + openbox + x11vnc, systemd-managed, independent of the real
monitor/GDM session) but simplified to one fixed-orientation display and a
custom camera-picker menu instead of tint2/browser/orientation-switching.

## Display

One new X display, `:2`, fixed at iPhone 14 portrait sizing — no
orientation-switching (unlike the sibling setup) since that wasn't asked
for here; can be added later with the same two-unit
`Conflicts=`/portrait+landscape pattern from
`headless-vnc-ipad-iphone-setup.md` if wanted.

| Display | Canvas | Resolution | VNC port |
|---|---|---|---|
| `:2` | Xvfb | 390x844x24 | 5901 |

## Packages to install

`xvfb`, `x11vnc`, `openbox` (`xterm` already present, useful for the menu's
debug/terminal entry; skip `tint2` — no taskbar needed for a single-purpose
camera desktop, keeping it genuinely simplified per the user's ask). None of
`xvfb`/`x11vnc`/`openbox` are installed on `nano-neil-2` yet (confirmed via
`dpkg -l` / `apt-cache policy` — all three are available candidates, none
installed).

## New systemd system units (root-owned, `/etc/systemd/system/`, same
pattern as the sibling machine)

- `xvfb-camera.service` — `Xvfb :2 -screen 0 390x844x24`, `User=neil`,
  `Restart=on-failure`, `ExecStartPre=-rm -f /tmp/.X2-lock`,
  `WantedBy=multi-user.target` (boots independent of GDM/login, same as
  sibling).
- `desktop-camera.service` — `After=`/`BindsTo=xvfb-camera.service`; runs
  `/home/neil/bin/start-camera-desktop.sh` (new script: `xsetroot -solid
  "#2c3e50"` for a non-black background, then `exec openbox --config-file
  /home/neil/.config/openbox-camera/rc.xml --menu-file
  /home/neil/.config/openbox-camera/menu.xml`).
- `x11vnc-camera.service` — `After=desktop-camera.service`; runs `x11vnc
  -display :2 -rfbport 5901 -forever -shared -usepw -rfbauth
  /home/neil/.vnc/passwd-camera`, `User=neil`, `Restart=on-failure`,
  `WantedBy=multi-user.target`. Needs a fresh password file —
  `x11vnc -storepasswd <password> /home/neil/.vnc/passwd-camera` (password
  value to come from the user at implementation time, not chosen
  unilaterally — this machine has no existing `/home/neil/.vnc/passwd` to
  reuse, unlike the sibling).

## Camera launcher + profile menu

- `/home/neil/bin/camera-launch.sh <args...>` — new wrapper: `pkill -f
  "simple_camera.py"; sleep 0.3` (releases the USB device from whatever's
  currently running), then `exec python3
  /home/neil/work/CSI-Camera/simple_camera.py "$@"` under `DISPLAY=:2`, so
  switching profiles is always a clean stop-then-start instead of a
  "device busy" error.
- `/home/neil/.config/openbox-camera/menu.xml` — custom root menu (same
  technique as the sibling's `~/.config/openbox/menu.xml` override), one
  entry per saved profile:
  - "Camera: MJPG (no profile)" → `camera-launch.sh usb`
  - "Camera: YUYV - defaults" → `camera-launch.sh usb --yuyv --profile /home/neil/work/camera/camera-settings/defaults.gpfl`
  - "Camera: YUYV - night" → `... night-defaults.gpfl`
  - "Camera: YUYV - zero-backlight" → `... zero-backlight.gpfl`
  - "Camera: YUYV - sunlight" → `... sunglight.gpfl`
  - "Camera: YUYV - backlight" → `... backlight.gpfl`
  - "Camera: YUYV - manual exposure" → `... manual-exp.gpfl`
  - "Stop Camera" → `pkill -f simple_camera.py`
  - "Terminal" (xterm, for debugging) — same rationale as the sibling's
    menu fix: use `xterm` directly rather than `x-terminal-emulator`, so it
    doesn't leak onto a different display's shared `gnome-terminal` server.
- `/home/neil/.config/openbox-camera/autostart` — Openbox autostart script
  (runs once when openbox starts, non-blocking): launches
  `camera-launch.sh usb --yuyv --profile .../backlight.gpfl &` as the
  default-on-boot profile (matches what's been tested live already this
  session), satisfying the original "runs at startup" ask while the menu
  above covers switching profiles afterward.
- `/home/neil/.config/openbox-camera/rc.xml` — copy of
  `/etc/xdg/openbox/rc.xml`; only needed if any margin/theme tweaks are
  wanted later (none required now — plain default is fine for a
  390x844 portrait canvas).

## Housekeeping

- `sudo systemctl daemon-reload`, then `enable --now` all three new units.
- No firewall step needed (`ufw` not installed on this machine, confirmed
  earlier).

## Verification

1. `systemctl status xvfb-camera desktop-camera x11vnc-camera` — all active.
2. `DISPLAY=:2 xrandr -q` → confirms `390x844` canvas.
3. `ss -tlnp | grep 5901` → `x11vnc` listening.
4. Connect RealVNC Viewer / Screens to `nano-neil-2.local:5901` (separate
   bookmark from the existing `:5900` GNOME share) — confirm the dark-blue
   Openbox desktop appears with the camera window already running
   (backlight profile, from autostart). Off the home network (cellular
   etc.), use the Tailscale address instead — see
   `tailscale-remote-access.md`.
5. Right-click → try at least two different profile menu entries in a row —
   confirm the old camera window/process cleanly stops and the new one
   starts with the new profile (no "device busy" errors).
6. "Stop Camera" then confirm no `simple_camera.py` process remains
   (`pgrep -af simple_camera.py`).
7. Confirm the real monitor session (`:1`, GNOME) and the existing 5900
   GNOME share (`vnc-screen-sharing.md`) are both unaffected throughout.
8. Full reboot with no monitor attached — confirm both VNC ports (5900 and
   5901) come up automatically with no console login, and the camera is
   already streaming on 5901 via the backlight-profile autostart.

## Keyboard input on the camera desktop - final resolved state (2026-08-29)

After a lot of back-and-forth (see the on-screen-keyboard entries above),
the root cause of "typing doesn't work" turned out to be **the Screens
iOS app itself getting into a stuck state** - its own native-keyboard
toolbar button stopped responding, unrelated to anything in this session.
Force-quitting and reopening Screens fixed it immediately. None of the
server-side keyboard work was the actual problem, but it's kept in place
as a useful fallback:

- **Primary**: Screens' own native keyboard (works normally; if it ever
  stops responding again, force-quit/reopen the app first before
  suspecting anything server-side).
- **Fallback**: `onboard`, toggle-only via the menu ("Toggle On-Screen
  Keyboard") - never auto-shown, confirmed to work mechanically
  end-to-end (a simulated tap correctly typed into the terminal); the
  earlier trouble with it looked like touch-precision on small keys
  rather than anything broken. Landscape orientation roughly doubles key
  size if that's ever worth trying.
- Both `open-camera-terminal.sh` and `open-claude-resume-terminal.sh`
  settled on `-bg black -fg white` for the terminal (readability), after
  briefly reverting and re-applying it while chasing what turned out to
  be the unrelated Screens app issue.
