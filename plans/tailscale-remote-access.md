# Remote access to nano-neil-2 via Tailscale

**Status: implemented, live-tested (2026-08-29).**

## Context

Both VNC services on `nano-neil-2` — the GNOME full-desktop share (port
5900, see `vnc-screen-sharing.md`) and the dedicated camera desktop (port
5901, see `usb-camera-autostart.md`) — were only reachable via
`nano-neil-2.local` or its LAN IP, which only work on the same local
network. The user wanted to reach the camera desktop from their phone off
that network (cellular data), which neither mDNS (`.local`) nor a raw LAN
IP support - neither routes over the internet.

Two options considered: port-forwarding 5901 directly to the internet
(rejected - VNC isn't designed to be safely internet-facing even with a
password), or a VPN mesh. Went with **Tailscale** - free for personal use,
no port forwarding or public exposure, works over WiFi or cellular from
anywhere once both devices are on the same tailnet.

## Setup performed

Installed via the official script (no `curl` on this machine, used `wget`
instead):
```
wget -qO- https://tailscale.com/install.sh | sh
sudo tailscale up --hostname=nano-neil-2
```
`tailscale up` requires an interactive browser login to authorize the
machine into the account - the user completed that step themselves rather
than through an agent-run command. Also installed the free Tailscale app
on the iPhone, signed into the same account.

## Current tailnet

| Device | Tailscale name | Tailscale IP |
|---|---|---|
| This machine | `nano-neil-2` | `100.115.68.112` |
| iPhone | `iphone-14` | `100.109.222.105` |

Confirmed via `tailscale status` - both devices show up in the same
tailnet.

## Connecting

From the iPhone, either VNC service now works from anywhere (WiFi or
cellular), not just the home network:
- Camera desktop: `100.115.68.112:5901` (or `nano-neil-2:5901` if
  MagicDNS is enabled on the tailnet)
- GNOME full-desktop share: `100.115.68.112:5900` (or `nano-neil-2:5900`)

The Tailscale IP is stable (doesn't change with DHCP the way the LAN IP
does), so it's a more durable address to bookmark than the LAN IP - though
`nano-neil-2.local` remains simpler when actually on the home network.

## Known non-fatal warning

`tailscale status` reports a health-check warning:
```
enabling connmark rules: ... iptables v1.8.7 (legacy): Couldn't load
match `connmark': No such file or directory
```
This is a missing kernel module specific to this Jetson's custom kernel
build, surfaced when Tailscale tries to set up connmark-based routing
rules. Didn't block basic peer-to-peer connectivity between the two
tailnet devices in testing - only worth investigating further if an
actual connection problem shows up (e.g. subnet routing, exit-node use).
