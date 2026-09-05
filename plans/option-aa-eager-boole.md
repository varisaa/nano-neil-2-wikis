# Switch nano-neil-2 to headless-by-default (Option A)

## Context
The user wants nano-neil-2 (Jetson Orin Nano, Ubuntu 22.04) to boot headless
by default to trim what starts up, while still being able to bring up the
full GNOME desktop on demand whenever a display is attached — without
rebooting each time.

Investigation this session confirmed:
- Current default systemd target: `graphical.target` (confirmed via
  `systemctl get-default`).
- The only things `graphical.target` adds on top of `multi-user.target` are:
  `gdm.service` (the login screen / desktop session), `accounts-daemon`,
  `apport`, `power-profiles-daemon`, `switcheroo-control`, `udisks2`, `lpd`,
  `kexec-load`/`kexec` (confirmed via `systemctl list-dependencies
  graphical.target`).
- The custom camera-desktop stack (`desktop-camera.service`,
  `x11vnc-camera.service`, `xvfb-camera-portrait.service`,
  `imx296-reload.service`) is `WantedBy=multi-user.target` and runs its own
  virtual (Xvfb) display — confirmed independent of `graphical.target` /
  physical display, so it keeps working unaffected in headless mode.
- `ssh.service` is enabled and running — remote access is available
  independent of the display, so switching to headless won't lock the user
  out.
- No custom `/etc/systemd/system/graphical.target.wants/` overrides beyond
  stock; this is a plain systemd target switch, no drop-in files to touch.

## Approach
Two commands, no reboot required, fully reversible at any time:

1. **Set the persistent default boot target to headless:**
   ```
   sudo systemctl set-default multi-user.target
   ```
   This only changes what boots next time (symlinks
   `/etc/systemd/system/default.target` → `multi-user.target.wants`
   equivalent); it does not stop anything running right now.

2. **Apply it immediately without rebooting**, so the current session
   reflects headless mode right away:
   ```
   sudo systemctl isolate multi-user.target
   ```
   This stops `gdm.service` and the other graphical-only units listed above.
   Camera-desktop/VNC and SSH stay up.

To bring the full desktop back later (e.g. a monitor gets plugged in), the
user runs:
```
sudo systemctl isolate graphical.target
```
and to drop back to headless again:
```
sudo systemctl isolate multi-user.target
```
(No plan changes needed for this — it's just documented as the ongoing
on-demand toggle, already covered in the prior conversation turn.)

## Notes / risks
- `systemctl isolate multi-user.target` will kill the current GNOME desktop
  session on `:0` if one is logged in and active. Since the user is
  interacting via this Claude Code console session (not the GNOME GUI
  session), this is expected and consistent with what they asked for.
- This is reversible instantly (`isolate graphical.target`), so no backup
  step is needed.

## Verification
After running both commands:
1. `systemctl get-default` → should print `multi-user.target`.
2. `systemctl status gdm.service` → should show `inactive (dead)`.
3. `systemctl status desktop-camera.service x11vnc-camera.service
   xvfb-camera-portrait.service` → should still show `active (running)`,
   confirming camera-desktop/VNC survived the switch.
4. `systemctl status ssh.service` → should still show `active (running)`.
5. Confirm the Jetson is still reachable over SSH/console (this session
   itself continuing to respond is sufficient evidence).
