# DRM device-busy error when switching multi-user → graphical.target

**Status: root-caused, not fixed (cosmetic, self-resolves - left as-is per user).**

## Context

While testing headless-by-default boot (`systemctl set-default
multi-user.target`, see the option-A headless plan under `work/plans/`),
switching back to the desktop with `sudo systemctl isolate
graphical.target` visibly showed an error on the physical monitor during
the transition. Initially suspected as a permissions/file-move failure,
but no matching error existed anywhere in the journal for that
description - that turned out to be unrelated pre-existing terminal
scrollback on a console session that became visible during the VT switch,
not a real fault.

The **actual** reproducible error, found in the `gdm-x-session` log for
every graphical-mode entry (e.g. boot at 13:03, gdm start at 13:08:57):

```
(EE) systemd-logind: failed to take device /dev/dri/card1: Device or resource busy
(EE) modeset(G0): drmSetMaster failed: Device or resource busy
(II) Failed to load module "modesetting" (already loaded, 0)
(EE) modeset(G0): drmSetMaster failed: Device or resource busy
```

## Root cause

This Jetson runs its own framebuffer compositor (`nvweston.service` /
`nvfb.service` / `nvfb-early.service`, all `WantedBy=multi-user.target`)
to render the console while in headless/text mode - it's not a plain
fbcon text console, it's a lightweight Weston session that holds the GPU
display device (`/dev/dri/card1`).

When switching to `graphical.target`, systemd stops those three units and
starts `gdm.service` (Xorg) at essentially the same second. systemd's
"Deactivated successfully" bookkeeping for `nvweston`/`nvfb` fires
slightly before the underlying DRM device is actually released at the
kernel/driver level, so Xorg's first `drmSetMaster` attempt(s) land in
that gap and fail with "Device or resource busy".

Confirmed via timestamps from the 13:03 reboot → 13:08:57 graphical
switch:

```
13:08:57  nvfb-early.service: Deactivated successfully
13:08:57  nvweston.service: Deactivated successfully
13:08:57  gdm.service ActiveEnterTimestamp
13:08:58  nvfb.service: Deactivated successfully
13:08:59  (EE) drmSetMaster failed: Device or resource busy   (x2, retried)
          [Xorg succeeds shortly after]
```

Confirmed **not fatal**: immediately after, `/dev/dri/card1` is held
cleanly and exclusively by Xorg (verified via `fuser`/`lsof`), and the
desktop comes up fully functional. It's a transient race during the
handoff, not a broken switch.

## Separately: the recurring benign gnome-session warning

Also present during every graphical startup, unrelated to the DRM race
above (this one's been seen on plain boots too, not just target
switches):

```
GnomeDesktop-WARNING: Could not create transient scope for PID <n>:
GDBus.Error:org.freedesktop.DBus.Error.UnixProcessIdUnknown: Process with
ID <n> does not exist.
systemd[...]: Failed to start Application launched by gnome-session-binary.
```

Same PID-already-exited-before-systemd-could-scope-it race seen at every
GNOME session start, retries and succeeds immediately after. Not
specific to the headless↔graphical switch.

## Fix considered, not applied

Could eliminate the DRM race by making `gdm.service` explicitly wait for
`nvweston.service`/`nvfb.service`/`nvfb-early.service` to fully stop
before starting (an `After=`/`Conflicts=` drop-in, possibly plus a short
`ExecStartPre=sleep`), rather than relying on systemd's near-simultaneous
stop/start ordering. Left alone for now since the switch already succeeds
on retry and the user prioritized other work over chasing a cosmetic
error.
