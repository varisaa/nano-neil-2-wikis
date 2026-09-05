# Display Mode menu silently did nothing when switching to desktop mode

**Status: root-caused and fixed.**

## Symptom

Clicking "Switch to Desktop Mode" in the openbox camera-desktop's Display
Mode menu (`display-mode-menu.sh`, reached via `:2`'s VNC) appeared to do
nothing — no error, no visible change, and (unlike the earlier "no
feedback" false alarm) the system genuinely stayed headless. The identical
`sudo systemctl isolate graphical.target` command, typed manually at the
physical console, worked immediately every time.

## Root cause

Openbox's `<action name="Execute"><command>...</command></action>` does
**not** run the command through a shell — it tokenizes the string on
whitespace only (quote-aware, but with no understanding of `;`, `&&`,
pipes, etc.) and execs argv[0] directly.

The menu's command had been chained to also re-fire the USB tether fix
after switching modes:
```
sudo systemctl isolate graphical.target; sudo /opt/nvidia/l4t-usb-device-mode/nv-l4t-usb-device-mode-state-change.sh
```

Openbox split this into argv as:
```
["sudo", "systemctl", "isolate", "graphical.target;", "sudo", "/opt/nvidia/.../nv-l4t-usb-device-mode-state-change.sh"]
```

`systemctl isolate` received `graphical.target;` (semicolon glued onto the
unit name) plus two extra unrecognized positional arguments, and rejected
them immediately — never asking systemd to transition at all. `sudo`
itself ran fine and logged a clean session open/close, which made this
look like it worked at first glance; only `systemctl` failed, silently,
with nothing surfacing on screen to show it.

Confirmed directly by comparing `gdm.service`'s own log against two failed
menu clicks:
```
00:23:44  gdm.service: Stopped                    (went headless)
00:24:29  [menu click] isolate graphical.target    -> gdm never tries to start
00:25:00  [menu click] isolate graphical.target    -> gdm never tries to start
00:25:09  [manual command, tty1]                   -> gdm starts 1 second later
```

## Fix

Removed the chaining entirely rather than wrapping it in `sh -c` — it was
made unnecessary by a separate, earlier fix:
`usb-tether-retrigger.service` (`WantedBy=multi-user.target` and
`graphical.target`, see the USB-tether/polkit investigation docs) now
re-checks the USB-C tether automatically as part of reaching either
target, regardless of what triggered the transition. So
`display-mode-menu.sh` went back to plain single commands:
```
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target
```

Verified live: switching to headless and back to desktop from the menu
now works reliably, with `gdm.service` starting immediately each time and
the USB tether surviving automatically via the systemd unit.

## Lesson for future menu items

Any future openbox `Execute` command needing more than one shell command
must be wrapped explicitly, e.g. `bash -c 'cmd1; cmd2'` — never rely on a
bare `;`/`&&` in the command string, since Openbox does not interpret
shell operators and will pass them through as literal characters,
producing a silent failure with no error surfaced anywhere.

## Related

- `wikis/investigations/camera-desktop-wifi-menu-polkit-denial.md` —
  separate, unrelated menu issue (network activation, not this one).
- `wikis/investigations/graphical-target-switch-drm-race.md` — a real but
  separate, self-resolving cosmetic DRM race during the same target
  transitions; not the cause of this issue.
