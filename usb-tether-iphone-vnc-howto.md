# Direct USB-C VNC from an iPhone to the camera desktop (2026-08-29)

No WiFi network required — just a USB-C cable, the Jetson's built-in USB
device-mode tether, and a VNC client on the phone. Confirmed working
end-to-end on `nano-neil-2`.

## How to connect

1. Plug a USB-C cable from the iPhone into the Jetson's USB-C **device
   mode** port (the same port used for flashing/serial console).
2. On the iPhone: **Settings → Wi-Fi → toggle on** (see note below — no
   network needs to be joined, no internet required).
3. iOS should show a "Linux for Tegra" entry under its network/Ethernet
   settings once the cable is connected — that confirms the link is up.
4. VNC to **`192.168.55.1:5901`** (same password as the regular camera
   desktop VNC). Any VNC client works.

## Why WiFi has to be toggled on (even with no network joined)

This was the confusing part. `192.168.55.1`/`l4tbr0` is a **completely
separate, self-contained subnet** with no code path shared with WiFi at
all (confirmed by reading every relevant service/config on the Jetson side)
— so this isn't a Jetson-side dependency.

It's an iOS-side behavior: with WiFi fully off, iOS won't route app traffic
(VNC included) over a secondary interface like USB-Ethernet, even though
the interface is fully up with a valid IP. Toggling WiFi on — without
joining any actual network — is enough to unblock it. Root mechanism
unconfirmed (can't inspect iOS internals), but the workaround is reliable
and still satisfies "no WiFi infrastructure needed in the field," since no
access point or internet connection is actually required.

## How the Jetson side works

- The USB-C port runs NVIDIA's `nv-l4t-usb-device-mode` stack — the Jetson
  presents itself as a USB **gadget** (peripheral), offering RNDIS + CDC-ECM
  network functions plus a mass-storage function. iOS recognizes the
  CDC-ECM function fine (shows as "Linux for Tegra" in network settings) —
  no host-mode compatibility problem, despite iPhones generally not
  supporting arbitrary USB network gadgets from other hosts.
- `l4tbr0` (bridge, `192.168.55.1/24`) and the private DHCP server that
  hands out the single fixed lease `192.168.55.100` are **not** the
  system's `isc-dhcp-server` (that stays disabled/unused) — they're driven
  entirely by udev:
  - `/etc/udev/rules.d/99-nv-l4t-usb-device-mode.rules` fires
    `nv-l4t-usb-device-mode-state-change.sh` on `usb_role`/`android_usb`
    subsystem change events (cable plug/unplug).
  - That starts/stops `nv-l4t-usb-device-mode-runtime.service`, which
    brings `l4tbr0` up and launches a private `dhcpd` (config regenerated
    each time at `/opt/nvidia/l4t-usb-device-mode/dhcpd.conf`, leases at
    `/run/l4t-usb-devmode-dhcpd.leases`).
  - Lease time is a deliberately short **15 seconds** — NVIDIA's documented
    workaround for host-side MAC-address randomization (so a single-address
    pool always reassigns the same IP regardless of which random MAC the
    host picks).

## Troubleshooting

If the cable is plugged in but `192.168.55.1` isn't reachable:

```bash
ip addr show l4tbr0          # expect: state UP, inet 192.168.55.1/24
ps aux | grep dhcpd          # expect: a dhcpd process bound to l4tbr0
```

If `l4tbr0` shows `state DOWN` and there's no `dhcpd` process, the udev
trigger likely didn't fire cleanly (observed once — a brief cable-wiggle or
role-switch hiccup tore the link down without a clean re-trigger). Fix by
manually re-running the same script udev would normally call:

```bash
sudo /opt/nvidia/l4t-usb-device-mode/nv-l4t-usb-device-mode-state-change.sh
```

This brought the link back within ~2 seconds with no reboot needed, and
the iPhone picked up `192.168.55.100` again immediately.

## Related

- `192.168.1.164` (fixed WiFi IP) is the alternative path when an actual
  WiFi network *is* available — see the camera-desktop VNC setup generally.
- Unrelated older setup for a different host: `plans/headless-vnc-ipad-iphone-setup.md`
  (iPad/iPhone virtual desktops on `ubuntu-neil`, a different machine).
