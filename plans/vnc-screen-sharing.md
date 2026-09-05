# GNOME full-desktop screen sharing on nano-neil-2

**Status: implemented, live-tested.**

## Context

The user wanted to see `nano-neil-2`'s desktop remotely from an iPad/iPhone
via a VNC-style app (RealVNC Viewer / Screens), the way it's already done
for a **different** machine (see
`wikis/plans/headless-vnc-ipad-iphone-setup.md`, which builds separate
per-device Xvfb virtual desktops on that machine). Exploration found none of
that infrastructure exists on `nano-neil-2` — no `xvfb-*`/`x11vnc-*` units,
no `/home/neil/bin/`, no `/home/neil/.vnc/`.

`nano-neil-2` already had a much simpler, native path available and unused
instead: GDM autologin is enabled (`AutomaticLoginEnable = true`,
`AutomaticLogin = neil`, Xorg not Wayland), so the real GNOME desktop on
`:1` comes up automatically at boot with no one physically present. GNOME's
own `gnome-remote-desktop` was already installed but its VNC backend was
disabled — its `grdctl` CLI can turn on screen-sharing of that real, live
`:1` session directly, standard VNC/RFB, no extra display server or window
manager involved.

## Setup performed

```
grdctl vnc enable
grdctl vnc set-auth-method password
grdctl vnc set-password '<password>'
grdctl vnc disable-view-only     # allow remote input, not just viewing
systemctl --user enable --now gnome-remote-desktop.service
```

- `set-auth-method password` matters for headless use — the default
  `prompt` mode requires someone physically at the machine to click
  "Accept" on each incoming connection.
- Default VNC port is 5900.
- No firewall present on this machine (`ufw` isn't even installed), so no
  port-opening step was needed.

## Connecting

RealVNC Viewer (iPad) or Screens (iPhone) → `nano-neil-2.local:5900`
(confirmed resolving correctly via mDNS to the machine's current LAN IP;
use the raw IP as a fallback if `.local` ever fails to resolve — check with
`hostname -I`, but note the DHCP lease can change, so prefer the hostname).
Only works on the same local network — see `tailscale-remote-access.md`
for reaching this from off-network (cellular, etc.).

This shares the **whole** real GNOME session (whatever's on the actual
monitor) — see `wikis/plans/usb-camera-autostart.md` for a separate,
simplified, camera-only Openbox desktop on its own port (5901), built
because the full desktop share was more than wanted for just watching the
camera.

## Verification

1. `grdctl status` — VNC backend enabled, auth-method `password`.
2. `ss -tlnp | grep 5900` — confirms `gnome-remote-desktop` listening.
3. Connect from the iPad (RealVNC Viewer) and iPhone (Screens) to
   `nano-neil-2.local:5900` — confirmed working live.
