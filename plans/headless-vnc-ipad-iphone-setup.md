# Headless VNC access for iPad + iPhone (independent virtual desktops)

## Result (2026-07-19) — implemented, working

The plan below was implemented with two changes discovered during
implementation, plus confirmed working end-to-end.

**Deviation 1 — dynamic xrandr resize doesn't work on this Xvfb build.**
Live-tested `xrandr --newmode`/`--addmode`/`-s` against a throwaway Xvfb
instance (package `xvfb 2:21.1.4-2ubuntu1.7~22.04.16`) and all three failed —
this Xvfb's RandR only reports the single mode it was started with. Switched
to: two separate systemd units per device (`xvfb-ipad-portrait.service` /
`xvfb-ipad-landscape.service`, same pattern for iphone), each hardcoding one
exact resolution via `-screen 0 WxHxDepth`, with `Conflicts=` between the
pair so starting one auto-stops the other. `set-orientation.sh` just does
`systemctl start xvfb-<device>-<orientation>.service` plus persists the
choice via enable/disable. This also meant the `cvt`/modeline work was
unnecessary — no CVT rounding-to-multiple-of-8 issue, exact pixel dimensions
(390x844 etc.) are used directly.
**Trade-off**: switching orientation restarts that device's Xvfb (a fresh X
session), so it briefly drops the VNC connection and resets any open
windows in that virtual desktop. openbox/tint2/x11vnc auto-reconnect within
~1-2s (`Restart=always` + a `wait-for-display.sh` poll loop), no manual
intervention needed.

**Deviation 2 — a real bug found via testing, fixed:** initial
`desktop-ipad.service`/`desktop-iphone.service` units had
`Wants=xvfb-ipad-portrait.service` (hardcoding portrait). Since openbox
loses its connection and gets restarted by systemd every time the
underlying Xvfb swaps, `Wants=` re-pulled portrait back up every time,
fighting the `Conflicts=` relationship and silently reverting any
landscape switch within ~2 seconds. Fixed by removing the `Wants=` line
(kept `After=` for both, ordering-only) — confirmed stable in both
directions afterward via a full switch/counter-switch/switch-back cycle
for both devices.

**Final layout** (all confirmed live):
| Device | Display | Port | Portrait | Landscape |
|---|---|---|---|---|
| iPad (5th gen) | `:2` | 5901 | 768x1024 | 1024x768 |
| iPhone 14 | `:3` | 5902 | 390x844 | 844x390 |

Verified:
- All 8 units (`xvfb-{ipad,iphone}-{portrait,landscape}`, `desktop-{ipad,iphone}`,
  `x11vnc-{ipad,iphone}`) installed under `/etc/systemd/system/`, enabled
  (portrait as the boot default for both), `daemon-reload`d.
- Ports 5901/5902 listening on all interfaces (`ss -tlnp`).
- `DISPLAY=:2`/`:3` `xrandr -q` shows exact requested resolutions.
- Orientation switch cycled ipad/iphone through landscape and back to
  portrait multiple times — stable, no ping-ponging after the `Wants=` fix.
- Real monitor session `:1` (GDM, `DP-1` 3440x1440) confirmed untouched and
  still active throughout — `systemctl is-active gdm.service` → active,
  `xrandr -q` on `:1` unchanged.
- Firewall (`ufw`) was already inactive, so no port rules were needed.
- Desktop shortcuts created at `/home/neil/Desktop/vnc-{ipad,iphone}-
  {portrait,landscape}.desktop`, marked `metadata::trusted` via `gio set`,
  mirrored into `~/.local/share/applications/` for the GNOME app grid. Each
  calls `sudo /home/neil/bin/set-orientation.sh <device> <orientation>`,
  permitted passwordlessly via a scoped `/etc/sudoers.d/vnc-orientation`
  rule (only those 4 exact invocations, and `set-orientation.sh` itself is
  root-owned/755 so `neil` can't edit what the NOPASSWD rule executes).
- Machine's LAN IP at time of setup: `192.168.1.249` (wlan `wlP1p1s0`) —
  connect RealVNC (iPad) to `192.168.1.249:5901`, Screens (iPhone) to
  `192.168.1.249:5902`. Existing `/home/neil/.vnc/passwd` reused for both.

**Not yet tested**: a full cold reboot with no monitor attached (to confirm
the two virtual desktops truly come up without any console login) — this
is disruptive to the user's active session so was left for the user to
run at a convenient time.

## Follow-up fixes (2026-07-19, same day, post-initial-verification)

Several real issues surfaced through actual phone/tablet usage after the
above was built. All fixed and verified live. **Machine is about to be
rebooted (with monitor attached this time) to test boot-time behavior —
this section is the "resume here" context for that.**

**1. iPhone showed a black screen after connecting.**
Not a bug — bare openbox+tint2 paints no wallpaper, so it really was just
solid black except a tiny tint2 bar easy to miss on a phone. Fixed by
adding `xsetroot -solid "#2c3e50"` to `/home/neil/bin/start-virtual-desktop.sh`
(runs before `tint2 &`/`exec openbox`). Confirmed via `scrot` screenshots on
both `:2` and `:3` — dark-blue background with taskbar clearly visible now.

**2. IP address changes on every DHCP renewal, breaking saved VNC bookmarks.**
Observed the LAN IP change three times in one session (`.249` → `.105` →
`.124`) purely from Wi-Fi DHCP churn. Fixed by:
- Renaming the host from `ubuntu` to **`ubuntu-neil`** via
  `hostnamectl set-hostname ubuntu-neil` (also updated the `127.0.1.1` line
  in `/etc/hosts`).
- Found `avahi-daemon` was already installed/active but was also
  advertising the `docker0` bridge interface, so `ubuntu-neil.local` could
  non-deterministically resolve to the unreachable `172.17.0.1` instead of
  the real Wi-Fi address. Fixed by adding `allow-interfaces=wlP1p1s0` under
  `[server]` in `/etc/avahi/avahi-daemon.conf`, then
  `systemctl restart avahi-daemon`. Verified 5x in a row resolving correctly.
- **Current connection addresses: `ubuntu-neil.local:5901` (iPad),
  `ubuntu-neil.local:5902` (iPhone)** — use the hostname, not an IP, from
  now on.

**3. Menu/launcher investigation surfaced a cross-display leak bug.**
Openbox's default right-click root menu's "Terminal emulator" item ran
`x-terminal-emulator` → `gnome-terminal`, which activates a single
per-user background server shared across the whole login session. Tested
live: launching it from the iPhone's virtual display (`:3`) actually
opened the new terminal window on the **real monitor session (`:1`)**,
invisible to the phone user. Fixed with a per-user
`/home/neil/.config/openbox/menu.xml` override that uses `xterm` (no
daemon/singleton, always honors its own `$DISPLAY`) for Terminal, and
`chromium-browser --new-window` for Web Browser. Verified `xterm` opens
correctly scoped to `:3`. Note: Chromium could theoretically hit a similar
issue if an instance is already running elsewhere and reuses its window via
the profile lock — not fixed (no repro yet), mention if it happens.
Also explained to the user what Reconfigure/Restart/Exit do in this
menu — tested "Exit" live by killing the tracked openbox PID: systemd's
`Restart=always` + default `KillMode=control-group` cleanly relaunches the
whole desktop (including tint2, no orphaned process) within ~1s. Confirmed
safe, not a real "logout."

**4. iPhone's rounded corners/notch/home-indicator masked window title-bar
close buttons** (screen renders edge-to-edge with no iOS safe-area
inset). Fixed with an iPhone-only Openbox config,
`/home/neil/.config/openbox/rc-iphone.xml` (full copy of
`/etc/xdg/openbox/rc.xml` with `<margins>` changed to
`top=40 bottom=40 left=20 right=20`, symmetric on purpose since it's
unverified how Screens maps portrait vs landscape rotation to the physical
notch/home-indicator edges). Wired in via
`start-virtual-desktop.sh`'s new optional 2nd arg → `openbox --config-file
<path>`; only `desktop-iphone.service`'s `ExecStart` passes it
(`/home/neil/bin/start-virtual-desktop.sh :3 /home/neil/.config/openbox/rc-iphone.xml`).
iPad's `desktop-ipad.service` deliberately left with no override (5th-gen
iPad has square corners, no notch/home-indicator, doesn't need the margin).
Verified: a freshly placed xterm window on `:3` now sits well clear of all
four edges with its titlebar/close button fully visible.

**5. "Web Browser" menu item didn't open Chromium — direct side-effect of
the hostname rename (item 2 above).** Chromium (snap) writes a
`SingletonLock` file recording `<hostname>-<pid>` when it starts. That lock
was created back when the machine was still called `ubuntu`
(`ubuntu-5268`). After renaming to `ubuntu-neil`, Chromium's singleton
check saw the lock's hostname didn't match the current one, concluded the
profile was "in use by another Chromium process on another computer," and
refused to launch at all (confirmed via direct CLI run, which surfaced the
exact error). PID 5268 was already dead — pure stale lock, not a real
conflict. Fixed by removing
`/home/neil/snap/chromium/common/chromium/Singleton{Lock,Socket,Cookie}`.
Verified: relaunched Chromium on `:3`, window opened correctly at
500x739 inside the margin-safe area, screenshot confirmed. Cleanly closed
afterward, no leftover processes.
**If Chromium ever again refuses to open with a "profile in use" message
(e.g. after another hostname change, or an unclean shutdown), this is the
fix** — remove those three Singleton* files while no chromium process is
actually running.

### Current file/state inventory (as of right before the monitor reboot test)
- `/home/neil/bin/start-virtual-desktop.sh` — now: `xsetroot` solid color,
  `tint2 &`, then `openbox` (with optional `--config-file $2`).
- `/home/neil/bin/set-orientation.sh`, `/home/neil/bin/wait-for-display.sh` — unchanged since initial build.
- `/home/neil/.config/openbox/menu.xml` — custom root menu (xterm + chromium).
- `/home/neil/.config/openbox/rc-iphone.xml` — iPhone-only margins config.
- `/etc/systemd/system/desktop-iphone.service` — updated `ExecStart` to pass the rc-iphone.xml config file; all other 7 units unchanged from initial build.
- `/etc/hosts`, hostname — now `ubuntu-neil` (was `ubuntu`).
- `/etc/avahi/avahi-daemon.conf` — added `allow-interfaces=wlP1p1s0`.
- All 8 systemd units were `active` and healthy immediately before this
  reboot: `xvfb-ipad-portrait`, `xvfb-iphone-portrait` (landscape variants
  enabled=disabled, the norm), `desktop-ipad`, `desktop-iphone`,
  `x11vnc-ipad`, `x11vnc-iphone`.

### What to check when back after this reboot (with monitor attached)
1. Monitor session (`:1`, GDM/GNOME on `DP-1`) boots normally, login screen, desktop as always.
2. `systemctl is-active xvfb-ipad-portrait xvfb-iphone-portrait desktop-ipad desktop-iphone x11vnc-ipad x11vnc-iphone` — all `active` without any manual intervention (this is the first time these units go through an actual boot, not just enable+start by hand).
3. Connect from iPad/iPhone to `ubuntu-neil.local:5901` / `:5902` — dark-blue desktop, taskbar visible, no black screen.
4. Confirm hostname survived reboot: `hostnamectl status` → `ubuntu-neil`.
5. If all good, the next untested milestone is a **fully headless boot (no monitor)** — not done yet, still pending.

## Context

The Jetson Orin Nano currently runs Ubuntu 22.04.5 with GDM3 + GNOME Shell on
session `:1`, driven by the Tegra `nvidia` driver bound to whatever monitor is
physically connected (currently `DP-1`, an ultrawide at 3440x1440). The user
wants to:

- Keep using the machine exactly as today when a monitor is attached (no
  change to the real GDM/GNOME session).
- Additionally get two dedicated, independently-sized virtual desktops —
  sized to match an iPad (5th gen) and an iPhone 14 — that come up
  automatically at boot and are reachable over VNC (RealVNC on iPad, Screens
  on iPhone) regardless of whether a monitor is attached or anyone has
  logged in physically.
- Be able to flip each of those two virtual desktops between portrait and
  landscape from desktop shortcuts.

Exploration confirmed two things that shape the approach:
- `x11vnc` is already installed and has been manually tested by the user
  against `:1` (password file already exists at `/home/neil/.vnc/passwd`,
  reusable as-is). No systemd unit or autostart entry exists yet for it.
- Jetson's Tegra display stack binds the `nvidia` driver via an `OutputClass`
  match in `/etc/X11/xorg.conf.d/tegra-drm-outputclass.conf`. Layering the
  generic `xserver-xorg-video-dummy` driver, or `nvidia-xconfig --virtual`,
  on top of that to fake an *extra* output alongside the real monitor is
  untested/uncertain territory on this platform, and risks disturbing the
  real `:1` session the user explicitly wants left alone.

Given the user confirmed they want the iPad/iPhone desktops to be fully
independent from the monitor session (not a mirror), the cleanest and safest
approach is to **not touch the GPU/monitor-bound `:1` session at all**, and
instead create two brand-new, pure-software X displays using `Xvfb` (virtual
framebuffer — no GPU/driver involvement, no interaction with GDM, no
autologin changes needed). Each gets its own lightweight window manager and
its own `x11vnc` instance on its own port, started by systemd at boot,
completely decoupled from monitor state or console login.

## Approach

### Layout
| Device | X display | Canvas (Xvfb) | Portrait mode | Landscape mode | VNC port |
|---|---|---|---|---|---|
| iPad (5th gen) | `:2` | 1024x1024x24 | 768x1024 | 1024x768 | 5901 |
| iPhone 14 | `:3` | 844x844x24 | 390x844 | 844x390 | 5902 |

Resolutions are iOS logical/"points" resolution (not native retina pixels),
matching what the user picked — far less VNC bandwidth, 1:1 with how iOS
actually lays out content.

The real monitor session (`:1`, GDM/GNOME, `nvidia` driver) is not modified
in any way.

### Components to install
- `xvfb` — provides the `Xvfb` binary.
- `openbox` — lightweight window manager for each virtual desktop.
- `tint2` — simple taskbar/panel so there's a usable launcher/switcher on
  each virtual desktop.
- `xterm` (or similar) so openbox's default root-menu "Terminal" entry works
  out of the box.
- `x11vnc` — already installed, reused as-is with the existing
  `/home/neil/.vnc/passwd`.

### Custom video modes
Xvfb starts with one canvas size but supports RANDR, so exact portrait/
landscape modes are added on top via `cvt` + `xrandr --newmode`/`--addmode`.
At implementation time, generate the 4 modelines with `cvt <w> <h> 60` (for
768x1024, 1024x768, 390x844, 844x390) and bake the resulting `Modeline`
strings into the orientation script below rather than recomputing them at
every switch.

### New files
- `/etc/systemd/system/xvfb-ipad.service`, `xvfb-iphone.service` — start
  `Xvfb :2 -screen 0 1024x1024x24` / `Xvfb :3 -screen 0 844x844x24`, running
  as `User=neil`, `Restart=on-failure`, with an `ExecStartPre=-rm -f` for
  stale `/tmp/.X{2,3}-lock` sockets, `WantedBy=multi-user.target` (no
  dependency on graphical.target/GDM at all).
- `/etc/systemd/system/desktop-ipad.service`, `desktop-iphone.service` —
  `After=`/`BindsTo=` the corresponding `xvfb-*` unit; runs a small wrapper
  script (`/home/neil/bin/start-virtual-desktop.sh <display>`) that sets the
  initial xrandr modes (adds all portrait/landscape modelines, applies
  portrait as the default), then execs `openbox` with `tint2 &` alongside it.
- `/etc/systemd/system/x11vnc-ipad.service`, `x11vnc-iphone.service` —
  `After=desktop-ipad.service`/`desktop-iphone.service`, runs
  `x11vnc -display :2 -rfbport 5901 -forever -shared -usepw -rfbauth /home/neil/.vnc/passwd`
  (and `:3`/`5902` for iPhone), `User=neil`, `Restart=on-failure`,
  `WantedBy=multi-user.target`.
- `/home/neil/bin/start-virtual-desktop.sh` — shared startup script
  (parameterized by display) that installs the custom modes via `xrandr` and
  launches openbox+tint2.
- `/home/neil/bin/set-orientation.sh` — shared script taking
  `ipad|iphone` + `portrait|landscape`, mapping to the right `DISPLAY` and
  `xrandr --output <output> --mode <mode>` call. This is what the desktop
  shortcuts invoke.
- Four `.desktop` launcher files in `/home/neil/Desktop/` (and mirrored into
  `~/.local/share/applications/` so they also show in the GNOME app grid):
  `iPad - Portrait.desktop`, `iPad - Landscape.desktop`,
  `iPhone - Portrait.desktop`, `iPhone - Landscape.desktop`, each calling
  `set-orientation.sh` with the right arguments. Note: GNOME/Nautilus will
  mark new desktop `.desktop` files as untrusted until the user right-clicks
  → "Allow Launching" once (or we set it programmatically via
  `gio set <file> metadata::trusted true`).

### Housekeeping
- First step: remove the stray `/home/neil/work/nvidia-xconfig-help.txt`
  file left behind by the read-only exploration (an agent slip that ran a
  shell redirect it shouldn't have).
- Copy this plan document into `/home/neil/work/ai-context/plans/` (create
  the `plans/` subdirectory if it doesn't exist yet) alongside the existing
  camera debugging docs, so it's kept as a persistent record of the setup
  the same way the camera work has been documented.
- Check `ufw status`; if the firewall is active, open TCP 5901/5902.
- `sudo systemctl daemon-reload`, then enable all 6 new units
  (`xvfb-ipad`, `xvfb-iphone`, `desktop-ipad`, `desktop-iphone`,
  `x11vnc-ipad`, `x11vnc-iphone`) with `systemctl enable --now`.

## Verification

1. `systemctl status xvfb-ipad xvfb-iphone desktop-ipad desktop-iphone x11vnc-ipad x11vnc-iphone` — all six active/running.
2. `DISPLAY=:2 xrandr -q` and `DISPLAY=:3 xrandr -q` — confirm the 4 custom
   modes exist and the expected portrait mode is applied by default.
3. Connect RealVNC (iPad) to `<jetson-ip>:5901` and Screens (iPhone) to
   `<jetson-ip>:5902` — confirm an openbox+tint2 desktop appears at the
   correct resolution on each.
4. From the real monitor's desktop, double-click each of the 4 new
   shortcuts one at a time and confirm the corresponding connected VNC
   client reflows to the expected orientation.
5. Reboot the whole machine with no monitor attached and no console login,
   then confirm from another machine that ports 5901/5902 come up and serve
   the right desktops automatically — this is the core "headless at boot"
   requirement.
6. Reattach the monitor, reboot again, and confirm `:1` still boots to the
   normal GDM login screen and GNOME desktop exactly as before — confirms
   the real desktop is unaffected (the "use it as I do today" requirement).
