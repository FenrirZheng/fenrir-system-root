# Mac-feel Keyboard on Linux (GNOME Wayland)

A reproducible recipe for getting macOS-style keyboard ergonomics on a Linux
desktop. Specifically targets a **GNOME Wayland session** on Debian 13
(`trixie`) with a PC105 keyboard, and uses [keyd](https://github.com/rvaiya/keyd)
+ [fcitx5](https://github.com/fcitx/fcitx5) as the only moving parts.

> Origin: built collaboratively in a single session on `2026-05-08` for
> [fenrir@fenrir](mailto:yy2025051204@gmail.com). This doc is the
> "knowledge layer" — config files, decisions, and the rationale behind them.
> It exists so a clean reinstall (or another Linux machine) can replay the
> setup without rederiving the design.

---

## 1. What this delivers

| Mac chord                              | Where it goes on this Linux |
|----------------------------------------|------------------------------|
| **Cmd + letter** (C/V/X/Z/A/T/W/Q/F/L/N/S/R/P) | `Ctrl + letter` (auto from `[cmd:C]` layer) |
| **Cmd + Left**                         | `Home` (line start) |
| **Cmd + Right**                        | `End` (line end) |
| **Cmd + Up**                           | `Ctrl+Home` (document start) |
| **Cmd + Down**                         | `Ctrl+End` (document end) |
| **Option + Left**                      | `Ctrl+Left` (jump one word back) |
| **Option + Right**                     | `Ctrl+Right` (jump one word forward) |
| **Option + Backspace**                 | `Ctrl+Backspace` (delete word back) |
| **Single tap LeftMeta (Win key)**      | `Super` alone -> GNOME Activities (~ Mission Control) |
| **Single tap LeftAlt**                 | `LeftAlt` alone (GUI menu mnemonic still works) |
| **CapsLock tap**                       | Toggle fcitx5 IM (English / Bopomofo chewing) |
| **CapsLock hold + key**                | `Ctrl + key` (preserve Linux/vim habit) |
| **Right Ctrl**                         | Plain Ctrl (no remap) |

What this does **not** deliver (deliberately removed during the session):

- `Cmd + Space` as Spotlight (was unreliable on GNOME-Wayland; user did not use it)
- `Cmd + Tab` as macOS app switcher (same reason)
- `Cmd + backtick` as same-app window cycle (same family, removed together)
- `Cmd + M` (minimize), `Cmd + H` (hide) — never wired up; can be added later

---

## 2. Architecture

Four layers, each owning exactly one concern:

```
+----------------------------------------------------------------+
| Physical keyboard (Logitech USB receiver + ASUS internal PC105)|
+--------------------------------+-------------------------------+
                                 | evdev events (KEY_* codes)
                                 v
+----------------------------------------------------------------+
|  keyd 2.5.0  (systemd: keyd.service, runs as root)             |
|  /etc/keyd/default.conf                                        |
|  - Maps physical keys to logical layers (cmd / opt / ctrl_layer)|
|  - Emits remapped events via /dev/uinput virtual keyboard      |
|  - For CapsLock-tap: forks /usr/local/bin/fcitx5-toggle        |
+--------------------------------+-------------------------------+
                                 | remapped evdev events
                                 v
+----------------------------------------------------------------+
|  Wayland compositor (GNOME mutter) + XWayland for X11 apps     |
|  Reads libinput -> applies xkb layout -> distributes to clients|
+--------------------------------+-------------------------------+
                                 | keysym events
                                 v
+------------------------------+---------------------------------+
|  GNOME Shell / mutter        |  Application clients            |
|  - Owns Super, Alt+Tab, etc. |  (Firefox, VSCode, terminal,    |
|  - GNOME Settings on F13/    |   gnome-text-editor, ...)       |
|    XF86Tools                 |                                 |
+------------------------------+---------------------------------+
                                              ^
                                              | DBus
+---------------------------------------------+-----------------+
|  fcitx5 (user systemd, autostarted via XDG)                   |
|  ~/.config/fcitx5/{profile,config}                            |
|  - Group "Default" with two IMs: keyboard-us + chewing        |
|  - Toggle via DBus (org.fcitx.Fcitx5)                         |
|  - Out-of-band toggle: /usr/local/bin/fcitx5-toggle invoked   |
|    by keyd, calls fcitx5-remote -t                            |
+---------------------------------------------------------------+
```

**Why this shape:** the toggle path (CapsLock -> fcitx5) deliberately bypasses
the keysym layer. Earlier attempts had keyd emit `F13` for fcitx5 to listen on,
but xkb maps `KEY_F13` -> `XF86Tools` keysym, and GNOME `media-keys` binds
`XF86Tools` to "open Settings" — so GNOME consumed the event before fcitx5
saw it. Routing through `fcitx5-remote -t` over DBus is a higher-level call
that no display-server layer can intercept.

---

## 3. Key decisions

### 3.1 Cmd-key location: physical LeftAlt becomes Cmd

`LeftAlt = overload(cmd, leftalt)` instead of mapping to LeftMeta(Win).
Rationale: on a PC105 keyboard the key adjacent to `space` is `LeftAlt`, which
corresponds to where Cmd lives on a Mac keyboard. The thumb falls naturally on
it. The LeftMeta(Win) key (one slot further out) is repurposed as Option (Mac
"alt"), so the relative ordering `Ctrl | Option | Cmd | Space` reading
left-to-right matches Mac.

Trade-off: `Alt-x` (Emacs `M-x`, GUI menu mnemonics) now lives on the LeftMeta
key. `overload(opt, leftmeta)` ensures a tap of LeftMeta still emits Super
alone (so GNOME Activities still works on a single Win-tap), only **hold** of
LeftMeta enters the `[opt:A]` layer.

### 3.2 Cmd-as-Ctrl mapping is global, no terminal exception

The user accepted that `Cmd+C` inside a terminal sends `Ctrl+C` (= SIGINT)
rather than copy. Copy in GNOME Terminal uses `Ctrl+Shift+C`, which from the
cmd layer is `Cmd+Shift+C`. The original physical LeftCtrl is unmodified, so
existing tmux `Ctrl-B` prefix (and any other Ctrl-anchored binding) keeps
working with the left pinky.

### 3.3 CapsLock is dual-purpose (tap-vs-hold)

`capslock = overload(ctrl_layer, command(/usr/local/bin/fcitx5-toggle))`:

- **Tap** (< ~200 ms) — fork the wrapper script, which calls
  `fcitx5-remote -t` over DBus. Toggles between English (`keyboard-us`) and
  Bopomofo (`chewing`).
- **Hold + key** — enters `[ctrl_layer:C]`, which auto-prefixes Ctrl on every
  key chord. Functionally identical to the user's old `capslock = leftcontrol`
  habit, so muscle memory survives.

This is *more* useful than the macOS original (where hold = Caps Lock toggle).
The Linux-style "hold = Ctrl modifier" behavior preserves vim/tmux/shell
ergonomics.

### 3.4 IME framework: fcitx5 (not ibus)

Both ibus and fcitx5 are installed on this machine. fcitx5 was chosen because
the chewing (Bopomofo) addon is only lightweight under fcitx5 in this distro
and the user already had `fcitx5-chewing` + `fcitx5-chinese-addons` installed
but unconfigured. ibus-daemon was running with no engines (the residual
default GNOME state).

Switching frameworks meant:

1. Killing `ibus-daemon` for the current session.
2. Setting `*_IM_MODULE=fcitx` env vars in
   `~/.config/environment.d/im-fcitx5.conf` (Wayland-friendly path; `~/.xprofile`
   is ignored by Wayland sessions — see Section 10.1).
3. Updating the running systemd user environment via
   `systemctl --user set-environment ...` so newly-launched apps see the new
   values immediately.
4. Adding `~/.config/autostart/org.fcitx.Fcitx5.desktop` so fcitx5 starts on
   next login.

ibus-daemon may still be spawned by GNOME Shell's built-in IBus integration
on subsequent logins (`--panel disable` mode for compose-key support). That's
fine — apps no longer talk to it because their `*_IM_MODULE` points to fcitx.

### 3.5 IME toggle: DBus call instead of synthetic keystroke

First attempted approach: `capslock = overload(ctrl_layer, f13)` and configure
fcitx5 `TriggerKeys=F13`. **Failed** because:

- `KEY_F13` -> X11 keycode 191 -> xkb maps to keysym `XF86Tools`
  (verified with `xmodmap -pk | grep ^191`)
- GNOME `org.gnome.settings-daemon.plugins.media-keys.control-center-static`
  binds `XF86Tools` to "open GNOME Settings"
- GNOME's grab is at the compositor layer, above fcitx5 — so fcitx5 never
  saw the event

The fix replaces the keystroke chain with a direct DBus call:

`/usr/local/bin/fcitx5-toggle` runs `fcitx5-remote -t` (which talks to
`org.fcitx.Fcitx5` over the user session bus). keyd's `command()` action
forks the script on tap; the script uses `runuser -u fenrir --` to drop
from root (keyd daemon's UID) to the user's UID with the right
`DBUS_SESSION_BUS_ADDRESS` and `XDG_RUNTIME_DIR` exported.

Bonus: future IME framework changes only require editing the wrapper script.
keyd config is decoupled from the IM identity.

---

## 4. File inventory

All paths absolute. Owner column shows who must own each file.

| Path | Owner | Purpose |
|---|---|---|
| `/etc/keyd/default.conf` | root:root | keyd config — physical-to-logical key map |
| `/etc/keyd/default.conf.bak.*` | root:root | Timestamped rollback points |
| `/usr/local/bin/fcitx5-toggle` | root:root, mode 0755 | DBus IME toggle wrapper, called by keyd |
| `/home/fenrir/.config/fcitx5/profile` | fenrir:fenrir | fcitx5 IM groups/items (keyboard-us + chewing) |
| `/home/fenrir/.config/fcitx5/config` | fenrir:fenrir | fcitx5 hotkeys (legacy F13 trigger left in but unused) |
| `/home/fenrir/.config/environment.d/im-fcitx5.conf` | fenrir:fenrir | IM env vars (Wayland-friendly) |
| `/home/fenrir/.config/autostart/org.fcitx.Fcitx5.desktop` | fenrir:fenrir | XDG autostart for fcitx5 |

GSettings (dconf) keys touched:

| Schema | Key | State |
|---|---|---|
| `org.gnome.shell.keybindings` | `toggle-application-view` | Reset to default `['<Super>a']` |
| `org.gnome.desktop.wm.keybindings` | `switch-input-source` | Explicit `@as []` (empty) |
| `org.gnome.desktop.wm.keybindings` | `switch-input-source-backward` | Explicit `@as []` |
| `org.freedesktop.ibus.general.hotkey` | `triggers` | Explicit `@as []` |

systemd state:

| Unit | State |
|---|---|
| `keyd.service` | enabled, active (replaces previously masked unit) |

---

## 5. Full file contents

### 5.1 `/etc/keyd/default.conf`

```
# Mac-feel layered keymap. Managed by Claude Code 2026-05-08.
# Rollback: sudo systemctl stop keyd  (or restore default.conf.bak.* in this dir)

[ids]
*

[main]
# Physical LeftAlt = Cmd. Hold = enter cmd layer; tap = leftalt (GUI menu mnemonic)
leftalt  = overload(cmd, leftalt)
# Physical LeftMeta(Win) = Option. Hold = opt layer; tap = super (GNOME Activities)
leftmeta = overload(opt, leftmeta)

# CapsLock: Mac-feel tap-vs-hold.
#   Tap  = run /usr/local/bin/fcitx5-toggle (DBus toggle, bypasses keysym chain)
#   Hold = ctrl_layer (auto-prefix Ctrl on every key chord)
capslock = overload(ctrl_layer, command(/usr/local/bin/fcitx5-toggle))

# RightCtrl: plain rightcontrol (no remap)

[ctrl_layer:C]
# Empty body. The :C suffix means "while in this layer, auto-prefix Ctrl on every key".

[cmd:C]
# Caret-movement overrides:
left  = home
right = end
up    = C-home
down  = C-end

[opt:A]
# Word-jump overrides:
left      = C-left
right     = C-right
backspace = C-backspace
```

### 5.2 `/usr/local/bin/fcitx5-toggle`

```bash
#!/bin/bash
# Toggle fcitx5 IM state. Called by keyd on CapsLock tap.
# keyd runs as root; this script drops to user fenrir's session bus
# so fcitx5-remote can reach the fenrir-owned fcitx5 daemon.
# Created by Claude Code 2026-05-08.
exec /usr/sbin/runuser -u fenrir -- env \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus \
  XDG_RUNTIME_DIR=/run/user/1000 \
  DISPLAY=:0 \
  /usr/bin/fcitx5-remote -t
```

Permissions: `chmod 755`, owner `root:root`. Hardcoded UID `1000` matches
fenrir's UID (`id fenrir`). If UID differs, edit accordingly.

### 5.3 `~/.config/fcitx5/profile`

```ini
[Groups/0]
# Group Name
Name=Default
# Layout
Default Layout=us
# Default Input Method
DefaultIM=keyboard-us

[Groups/0/Items/0]
# Name
Name=keyboard-us
# Layout
Layout=

[Groups/0/Items/1]
# Name
Name=chewing
# Layout
Layout=

[GroupOrder]
0=Default
```

### 5.4 `~/.config/fcitx5/config`

```ini
[Hotkey]
# Trigger key: F13 emitted by keyd from CapsLock tap (Mac-feel toggle).
# NOTE: This was the original design. Replaced by DBus-toggle path; left in
# the file as a harmless fallback.
TriggerKeys=F13
EnumerateWithTriggerKeys=True
EnumerateForwardKeys=
EnumerateBackwardKeys=
EnumerateGroupForwardKeys=
EnumerateGroupBackwardKeys=
ActivateKeys=
DeactivateKeys=
PrevPage=Up
NextPage=Down
AltTriggerKeys=

[Behavior]
ActiveByDefault=False
ShareInputState=No
PreeditEnabledByDefault=True
ShowInputMethodInformation=True
ShowInputMethodInformationWhenFocusIn=False
CompactInputMethodInformation=True
ShowFirstInputMethodInformation=True
DefaultPageSize=5
OverrideXkbOption=False
CustomXkbOption=
EnabledAddons=
DisabledAddons=
PreloadInputMethod=True
AllowInputMethodForPassword=False
ShowPreeditForPassword=False
AutoSavePeriod=30
```

### 5.5 `~/.config/environment.d/im-fcitx5.conf`

```ini
# IM framework selection. Read by systemd user instance at session start.
# Wayland-friendly entry point (replaces ~/.xprofile which only X11/GDM sources).
# Created by Claude Code 2026-05-08.
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
GLFW_IM_MODULE=ibus
```

`GLFW_IM_MODULE=ibus` is intentional — GLFW (used by some game-dev tools) only
recognizes `ibus` as a value, but actually still routes through the system IM
framework, which is now fcitx. This keeps GLFW happy without breaking IM.

### 5.6 `~/.config/autostart/org.fcitx.Fcitx5.desktop`

```ini
[Desktop Entry]
Name=Fcitx 5
GenericName=Input Method
Comment=Start Input Method
Exec=/usr/bin/fcitx5
Icon=fcitx
Terminal=false
Type=Application
Categories=System;Utility;
StartupNotify=false
X-GNOME-Autostart-enabled=true
X-GNOME-Autostart-Phase=Initialization
```

---

## 6. Setup-from-scratch procedure

Assumes a fresh Debian 13 GNOME Wayland install on `/home/fenrir` (UID 1000).

### 6.1 Install packages

```bash
sudo apt install -y keyd fcitx5 fcitx5-chewing fcitx5-chinese-addons
```

### 6.2 Drop in keyd config

Copy Section 5.1 contents to `/etc/keyd/default.conf` via `sudo tee`, then:

```bash
sudo systemctl unmask keyd 2>/dev/null || true
sudo systemctl enable --now keyd
sudo systemctl status keyd --no-pager
```

Verify in journalctl that the device match list includes the user's keyboard:

```bash
journalctl -u keyd --no-pager | rg 'DEVICE: match'
```

### 6.3 Drop in fcitx5 toggle wrapper

Copy Section 5.2 contents to `/usr/local/bin/fcitx5-toggle` via `sudo tee`,
then `sudo chmod 755 /usr/local/bin/fcitx5-toggle`.

### 6.4 Drop in fcitx5 user config (run as fenrir)

```bash
mkdir -p ~/.config/fcitx5/conf
mkdir -p ~/.config/environment.d
mkdir -p ~/.config/autostart
```

Then write Sections 5.3 / 5.4 / 5.5 / 5.6 to their paths.

### 6.5 Update current systemd user environment (no logout needed)

```bash
systemctl --user set-environment GTK_IM_MODULE=fcitx QT_IM_MODULE=fcitx \
  XMODIFIERS=@im=fcitx SDL_IM_MODULE=fcitx
```

### 6.6 Stop ibus, start fcitx5

```bash
pkill -TERM ibus-daemon  # may restart automatically; harmless
fcitx5 -d &
fcitx5-remote            # should print 1 or 2; not abort
```

### 6.7 Clean up gsettings collisions

```bash
gsettings set org.gnome.desktop.wm.keybindings switch-input-source "@as []"
gsettings set org.gnome.desktop.wm.keybindings switch-input-source-backward "@as []"
gsettings set org.freedesktop.ibus.general.hotkey triggers "@as []"
```

After full setup, **log out and back in once** so all newly-launched apps
inherit the correct env. Already-running apps (Firefox, VSCode, etc.) still
hold the old `IM_MODULE` until restarted.

---

## 7. Verification matrix

Run each test in a freshly-launched app (existing apps may have stale env).

| # | Action | Expected | Component verified |
|---|---|---|---|
| 1 | `Cmd+T` in Firefox | New tab | `[cmd:C]` letter chord auto-Ctrl |
| 2 | `Cmd+W` in Firefox | Close tab | same |
| 3 | `Cmd+Left` in any text input | Cursor jumps to line start | `[cmd:C] left = home` |
| 4 | `Option+Left` in any text input | Cursor jumps one word back | `[opt:A] left = C-left` |
| 5 | Single-tap LeftMeta (Win) | GNOME Activities Overview | `overload(opt, leftmeta)` tap-emit |
| 6 | RightCtrl + C in terminal | SIGINT | RightCtrl pass-through |
| 7 | Hold CapsLock + V in editor | Paste | `[ctrl_layer:C]` auto-Ctrl |
| 8 | Tap CapsLock once | `fcitx5-remote` flips 1<->2; chewing IM panel appears | command() -> fcitx5-toggle -> DBus |
| 9 | Tap CapsLock, type letters in fresh GTK app | Bopomofo prompts (ㄅㄆㄇㄈ) appear | fcitx5 + chewing addon |
| 10 | Tmux `Ctrl-B` (physical LeftCtrl) | Tmux prefix triggered | LeftCtrl pass-through |

If 1-7 work but 8 doesn't: see Section 9.4 (DBus-toggle path debugging).
If 8 works but 9 doesn't: see Section 9.5 (chewing addon not loaded).

---

## 8. Rollback procedure

### 8.1 Full disable (rebuild from scratch needed afterward)

```bash
sudo systemctl stop keyd
sudo systemctl mask keyd                        # prevents next-boot autostart
gsettings reset org.gnome.shell.keybindings toggle-application-view
gsettings reset org.gnome.desktop.wm.keybindings switch-input-source
gsettings reset org.gnome.desktop.wm.keybindings switch-input-source-backward
gsettings reset org.freedesktop.ibus.general.hotkey triggers
rm ~/.config/environment.d/im-fcitx5.conf
rm ~/.config/autostart/org.fcitx.Fcitx5.desktop
pkill fcitx5
```

After log-out / log-in, system reverts to factory GNOME behavior.

### 8.2 Per-layer rollback

| Layer broken | Action |
|---|---|
| keyd config corrupted | `sudo cp /etc/keyd/default.conf.bak.<TS> /etc/keyd/default.conf && sudo systemctl restart keyd` |
| keyd service crashing | `sudo systemctl stop keyd` (instantly back to native keyboard) |
| fcitx5 not toggling | `pkill fcitx5; fcitx5 -d &` (force re-read of profile/config) |
| fcitx5 toggle wrapper broken | Run `/usr/local/bin/fcitx5-toggle` manually as root, observe error; check `runuser` path, DBus address, fcitx5-remote installed |
| GNOME shortcut conflict | `gsettings reset <schema> <key>` for the offending binding |

---

## 9. Troubleshooting

### 9.1 keyd starts but my keyboard isn't remapped

`journalctl -u keyd | rg 'DEVICE: (match|ignoring)'` shows what keyd grabbed.
External Bluetooth keyboards may not appear immediately — restart keyd after
pairing: `sudo systemctl restart keyd`.

### 9.2 Cmd+letter sends Ctrl+Alt+letter (or extra modifiers)

The `[cmd:C]` layer requires the `:C` suffix to auto-prefix Ctrl. Missing
suffix produces literal Cmd+letter without Ctrl translation. Confirm config
file matches Section 5.1 exactly (mind the colon and the capital C).

### 9.3 LeftAlt-tap doesn't activate GUI menus

`overload(cmd, leftalt)` should emit `leftalt` on tap. If menus don't open on
single Alt-tap, the tap window may be exceeded by the user's natural press
duration. Tune with `overloadt(cmd, leftalt, 200)` (200 ms) — increase the
number to allow longer "taps".

### 9.4 CapsLock-tap doesn't toggle IM (DBus path)

Run the wrapper directly to see errors:

```bash
/usr/local/bin/fcitx5-toggle
echo $?
```

Common causes:

- `runuser: invalid user` -> verify `id fenrir` returns UID 1000; otherwise
  hardcoded path `/run/user/1000/bus` is wrong.
- `Failed to create dbus connection` -> fcitx5 isn't running. Start it:
  `pgrep -x fcitx5 || (fcitx5 -d &)`.
- `command not found: runuser` -> install `util-linux` (should be present by
  default on Debian); update wrapper to use full path `/usr/sbin/runuser`.

### 9.5 fcitx5 toggles state but typing doesn't show Bopomofo

Open `fcitx5-configtool` (GUI) and confirm the chewing engine appears in
"Current Input Method" group. If not, click `+` -> search "chewing" -> add.
Also verify `~/.config/fcitx5/profile` has chewing as `Items/1`.

### 9.6 Already-running apps still don't see fcitx after env update

Process env is forked at start time and never refreshed. `systemctl --user
set-environment ...` only affects future child processes. **Restart the app**
(close and reopen). Logging out completely is the surest way.

### 9.7 GNOME Settings UI opens when I tap CapsLock

You're on an older version of this setup where keyd emitted `KEY_F13`. xkb
maps that to `XF86Tools` keysym, and GNOME binds `XF86Tools` to "open
Settings". Replace `capslock = overload(ctrl_layer, f13)` with
`capslock = overload(ctrl_layer, command(/usr/local/bin/fcitx5-toggle))` per
Section 5.1. The DBus-toggle path bypasses the keysym chain entirely.

---

## 10. Lessons learned (the gotchas)

These are the surprising bits a future setup will trip over.

### 10.1 `~/.xprofile` is dead under Wayland

`~/.xprofile` is sourced by GDM's X11 session script. **Wayland sessions skip
it entirely.** The Wayland-friendly equivalent is `~/.config/environment.d/`
(systemd user environment generator). Files there must end in `.conf` and
contain `KEY=VALUE` lines (no `export`, no shell logic — they're parsed by
systemd, not bash).

### 10.2 `gsettings reset` is not "make this empty"

`gsettings reset <schema> <key>` resets to the **schema-defined default**, not
to empty. The default is whatever the distro maintainer wrote into the
`.gschema.xml`. To force empty, use the explicit form:

```bash
gsettings set org.example.schema my-key "@as []"
```

The `@as` is a GVariant type hint; without it, dconf can't always serialize an
empty array. Most array-of-strings keys need this hint.

### 10.3 Root-shell `HOME=/root` trap

If Claude Code (or any shell session) is started under `sudo`, `$HOME` is
`/root`, **not** the invoking user's home. Commands like `mkdir -p ~/.config/x`
silently land in `/root/.config/x` instead of where you wanted. Either:

- Use absolute paths (`/home/fenrir/.config/x`) in shell commands, or
- Prefix with `runuser -u fenrir --` to explicitly drop privileges.

The `Write` tool in Claude Code accepts absolute paths so it doesn't trip
on this — only `Bash` shell-expanded `~` does.

### 10.4 GNOME `media-keys` schema is a busy keysym graveyard

Many "free-looking" keysyms are pre-claimed by the
`org.gnome.settings-daemon.plugins.media-keys` schema. Notably:

- `XF86Tools` -> open Settings (control-center)
- `XF86Calculator` -> calculator
- `XF86Search` -> search
- `XF86Mail` -> email client

…and there are ~80 such bindings out of the box. KEY_F13 happens to map to
`XF86Tools` keysym on Debian's default xkb evdev layout, which is the head-on
collision that broke the first design. Lookup before picking a synthetic key:

```bash
gsettings list-recursively org.gnome.settings-daemon.plugins.media-keys
```

### 10.5 fcitx5 over XWayland needs the `xcb` AND `waylandim` addons

Most apps under GNOME-Wayland are XWayland (Firefox, electron apps). Some
are native Wayland (gnome-text-editor, GTK4 apps in newer mode). fcitx5 uses
its `xcb` addon for the former and `waylandim` for the latter. Both are loaded
by default when fcitx5 starts under a Wayland-Server-with-XWayland session;
verify with `journalctl --user -u fcitx5 | rg 'Loaded addon'`.

### 10.6 `IM_MODULE=ibus` lingers in systemd user env

After `pkill ibus-daemon`, the `QT_IM_MODULE=ibus` and `XMODIFIERS=@im=ibus`
values still live in the systemd user environment (set by GNOME at login from
some default config). Newly-launched apps inherit those — and try to talk to
the dead ibus-daemon. Always update systemd's env with
`systemctl --user set-environment ...` after switching frameworks, **then**
restart your apps.

---

## 11. Maintenance

| Trigger | Check |
|---|---|
| Kernel upgrade | keyd uses uinput; should keep working. If keyboard "doesn't respond" after boot, `sudo systemctl status keyd`. |
| GNOME version bump | `gsettings list-recursively org.gnome.shell.keybindings` to see if any new defaults conflict with cmd-layer overrides. |
| fcitx5 version bump | `~/.config/fcitx5/config` schema may add fields; let fcitx5 self-extend (it preserves existing keys). |
| New keyboard | `journalctl -u keyd` should show it matched. If not, restart keyd. |
| User UID change | Update hardcoded `1000` in `/usr/local/bin/fcitx5-toggle`. |

---

## 12. Reference

- keyd man page: `man keyd`, `man keyd.conf`
- keyd source: <https://github.com/rvaiya/keyd>
- fcitx5 wiki: <https://fcitx-im.org/wiki/Fcitx_5>
- fcitx5 chewing addon: <https://github.com/fcitx/fcitx5-chewing>
- GNOME settings-daemon media-keys schema: `gsettings list-recursively org.gnome.settings-daemon.plugins.media-keys`
- xkb evdev keycode table: `/usr/share/X11/xkb/keycodes/evdev`
- systemd environment generators: `man systemd.environment-generator`
