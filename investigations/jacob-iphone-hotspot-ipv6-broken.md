# Claude Code stalls on "Waiting for API response" while on Jacob-Work-iPhone hotspot

**Status: root-caused and fixed.**

## Symptom

This Claude Code session (running directly on `nano-neil-2`) intermittently
showed `Waiting for API response · will retry in 1m 36s` while the
Jetson's WiFi was connected to `Jacob-Work-iPhone` (iPhone Personal
Hotspot). No such stalls on `varisaa-fiber`.

## Root cause

The hotspot's IPv6 is broken. Confirmed directly:

```
$ ip -6 -o addr show wlP1p1s0
    inet6 2600:1010:...:1ed0/64 scope global temporary tentative dynamic
    inet6 2600:1010:...:6b9d/64 scope global tentative mngtmpaddr noprefixroute
```

Both assigned IPv6 addresses stayed stuck `tentative` — duplicate address
detection (DAD) never completed, meaning the kernel considers them
provisional/unusable even though they're present.

Direct proof the path is actually dead, not just cosmetically "tentative":

```
IPv4 to api.anthropic.com (160.79.104.10:443) → connected fine
IPv6 to api.anthropic.com (2607:6bc0::10:443) → "No route to host"
```

`getent hosts api.anthropic.com` returns only the `AAAA` (IPv6) record by
default on this system's resolver, and address-selection prefers IPv6 when
an address is configured on the interface — so any client resolving and
connecting normally (this Claude Code session included) would try the
dead IPv6 path first and have to time out before any IPv4 fallback,
exactly matching the observed multi-minute retry stalls.

This is a hotspot-side issue (how the iPhone hands out/routes IPv6 to
tethered WiFi clients), not something wrong with the Jetson's own network
stack — `varisaa-fiber`'s IPv6 works fine (confirmed working IPv6 DNS/
routing on that connection separately, no stalls there).

## Fix

Disabled IPv6 on just the `Jacob-Work-iPhone` NetworkManager connection
profile, so nothing on this box can get stuck trying it over that
specific network:

```
sudo nmcli connection modify 'Jacob-Work-iPhone' ipv6.method disabled
sudo nmcli connection up 'Jacob-Work-iPhone'
```

Scoped to this one profile only — `varisaa-fiber` and any other saved
network are untouched. Verified after applying: no more IPv6 addresses on
`wlP1p1s0` while connected to this profile, and a fresh TCP connect to
`api.anthropic.com:443` succeeds immediately (goes straight to IPv4, no
stall).

**If this connection profile is ever deleted and recreated** (e.g. by
reconnecting via the WiFi menu's password-prompt flow again), this
setting will need to be reapplied — it lives on the profile, not
globally.

## Related

- `wikis/investigations/camera-desktop-wifi-menu-polkit-denial.md` — the
  earlier fix that let the openbox WiFi menu activate this profile at all
  (separate polkit issue, unrelated to this one).
- `wikis/usb-tether-iphone-vnc-howto.md` — a different, unrelated iPhone
  connectivity path (USB tether, not this WiFi hotspot).
