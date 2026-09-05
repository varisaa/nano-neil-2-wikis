# Orientation switch: VNC client shows new geometry with stale cursor mapping

**Status: root-caused. Server-side behavior confirmed correct; the
mismatch is VNC client-side reconnect behavior, not fixable from
nano-neil-2.**

## Symptom

After switching orientation via the camera-desktop menu (Orientation:
Portrait / Landscape → `set-camera-orientation.sh`), the VNC connection
dropped, then came back showing the new orientation's canvas — but mouse/
touch input behaved as if the screen were still the OLD orientation
("landscape mode with portrait mode cursor and experience"): clicks landed
in the wrong place relative to what was drawn.

## Root cause

Orientation switch is architecturally a full X-server replacement, not a
resize: `set-camera-orientation.sh` stops the old-orientation
`xvfb-camera-{portrait,landscape}.service` and starts the other one — a
brand new `Xvfb` process on `:2` at the new pixel dimensions (they
`Conflicts=` each other, confirmed via
`systemctl show ... -p Conflicts`). This is deliberate; Xvfb on this
system doesn't support live RandR resizing (documented already in
`wikis/plans/headless-vnc-ipad-iphone-setup.md`).

Killing the old Xvfb kills `x11vnc-camera.service`'s connection to it too.
Confirmed directly in the journal at the moment of an orientation switch:

```
13:05:48  x11vnc[1691]: caught XIO error
13:05:48  x11vnc-camera.service: Main process exited, code=exited, status=3/NOTIMPLEMENTED
13:05:48  x11vnc-camera.service: Failed with result 'exit-code'.
13:05:50  x11vnc-camera.service: Scheduled restart job, restart counter is at 3.
13:05:50  Started x11vnc server for the camera virtual display (:2, port 5901).
```

`Restart=always` on the service (confirmed via `systemctl show`) brings a
fresh `x11vnc` process up ~2 seconds later, correctly attached to the new
Xvfb, listening again on 5901 with no server-side errors. This part works
exactly as designed.

The mismatch is what happens on the **client** side of that drop. A
proper VNC reconnect re-does the full protocol handshake (`ServerInit`
message, which carries the current width/height) and should always learn
the new geometry correctly. If a VNC viewer app instead treats the broken
socket as a transient blip and silently retries/resumes rather than doing
a full fresh handshake, it can end up rendering the new framebuffer's
pixels (genuinely landscape) while still using cached UI chrome / input
coordinate mapping from the old orientation (portrait) — exactly the
observed symptom. This is viewer-app reconnect logic, entirely outside
what runs on nano-neil-2.

## Not the cause (ruled out)

- x11vnc IPv6 listener: after a restart, `listen6: bind: Address already
  in use` appears briefly (old process's socket not yet released) and it
  falls back to IPv4-only — but the connected client in this case was
  already IPv4 (`192.168.55.100`, the USB tether address) throughout, so
  this wasn't a factor here. Could matter for an IPv6-connected client
  specifically; not confirmed either way.
- The `set-camera-orientation.sh` script itself: runs cleanly, confirmed
  via direct manual invocation, sudo invocation, and a simulated
  sessionless-context invocation (same technique used for the WiFi-menu
  polkit investigation) — all three succeeded with correct systemd state
  transitions (`Conflicts=` correctly stopping the other orientation,
  target service starting cleanly).

## Workaround

After switching orientation, **manually disconnect and reconnect** the
VNC client rather than trusting its auto-reconnect — a fresh, deliberate
connection always does the full handshake and picks up the correct
geometry. No Jetson-side fix applies since the bug (if it can be called
that) lives in the client's reconnect logic, which varies by VNC app.

## Related

- `wikis/plans/headless-vnc-ipad-iphone-setup.md` — documents the
  Xvfb-can't-resize-live design decision this behavior stems from (a
  different machine, same underlying pattern).
- `wikis/investigations/display-mode-menu-semicolon-noop.md` — a separate,
  unrelated menu bug found the same night.
