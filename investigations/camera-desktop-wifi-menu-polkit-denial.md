# Camera-desktop WiFi menu fails with "Not authorized to control networking"

**Status: root-caused, fix designed, not yet applied (blocked by the
permission classifier - needs explicit user approval to write the polkit
rule and restart `polkit.service`).**

## Symptom

Clicking a network in the openbox camera-desktop's WiFi submenu
(`list-wifi-menu.sh`, `:2`) silently fails to connect — no visible error on
screen, connection just never activates. Reproduced repeatedly against
`Jacob-Work-iPhone` (a saved Bluetooth... no, WiFi hotspot connection
profile), even though the exact same `nmcli connection up` command
succeeds instantly when run from a normal shell.

## Root cause

`journalctl -u NetworkManager` shows the real error on every failed click:

```
audit: op="connection-activate" uuid="67c9afcf-..." name="Jacob-Work-iPhone"
pid=<nmcli pid> uid=1000 result="fail" reason="Not authorized to control networking."
```

This is a **polkit** denial, not a NetworkManager or menu-script bug.

`desktop-camera.service` (the systemd unit that runs the Xvfb + openbox
session on `:2`, `User=neil`) has `PAMName=` unset — it never registers a
real PAM/logind session. Confirmed via `loginctl list-sessions`: only the
two genuine logins show up (console `ttyTCU0`, physical `seat0` tty2/GNOME)
— nothing for `:2`.

`pkaction --action-id org.freedesktop.NetworkManager.network-control
--verbose` shows:
```
implicit any:      auth_admin
implicit inactive: yes
implicit active:   yes
```

Polkit's active/inactive-session auto-allow rules only apply when the
D-Bus caller can be resolved to an actual logind session. Since `:2`'s
processes have none, polkit can't match "active" or "inactive" and falls
back to `implicit any: auth_admin` — which requires an authentication
agent to prompt for a password. No such agent exists in the bare
Xvfb+openbox `:2` session, so the request just fails outright with no
prompt and no visible error.

This affects **every** WiFi menu action that activates a connection (both
the saved-network quick-connect items and the secured-network password
flow via `wifi-connect-secured.sh`), not just this one network — any
earlier apparent "success" clicking a saved network was actually coming
from a different, properly-sessioned context (a real shell, or the actual
GNOME desktop on seat0), not the `:2` menu itself.

## Fix designed, not applied

A scoped polkit rule granting `network-control` directly to `neil`,
regardless of session state:

```js
// /etc/polkit-1/rules.d/50-neil-network-control.rules
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.NetworkManager.network-control" &&
        subject.user == "neil") {
        return polkit.Result.YES;
    }
});
```

Then `systemctl restart polkit.service` to pick it up.

**This was blocked by Claude Code's own auto-mode permission classifier**
(writing system-wide polkit authorization policy + restarting a security
daemon is treated as sensitive) — needs the user to explicitly approve
running it.

### Alternative considered, rejected as more invasive

Give `desktop-camera.service` a real PAM session instead (`PAMName=` +
a PAM stack config), so it shows up as a proper logind session and
polkit's normal active-session auto-allow applies naturally. More
"textbook correct," but changes more surface area than the actual problem
warrants: the service stops being a plain background unit and becomes a
tracked login session (own logind session ID, different cgroup/scope
semantics, shows up in `loginctl`, could interact with idle/lock/suspend
logic elsewhere on the box) - larger blast radius for a fix that's really
about one user being denied one specific action. The polkit-rule approach
gets the same practical outcome with a single, easily-reversible file.

## Next step

Apply the polkit rule above (with user approval), restart `polkit.service`,
then have the user re-test the WiFi menu from `:2` to confirm the fix.
