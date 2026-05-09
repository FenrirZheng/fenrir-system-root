# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **system-config repo whose working tree IS the filesystem root** (`/`), but whose `.git` lives at `$HOME/.sysconfig.git` as a bare repo. It tracks a tiny set of system files under `/etc` and `/usr/local/bin` that implement a Mac-feel keyboard layer for a Debian 13 GNOME-Wayland desktop (user `fenrir`, UID 1000).

Edits operate against live system files. Changes to tracked paths take effect on the running system the moment they are written — there is no build step and no staging environment.

## Bare-repo layout

There is **no `.git` directory at `/`** and **no `.gitignore`**. Instead, all git state is at `/root/.sysconfig.git` (bare), and `/root/.bashrc` defines:

```
alias sc='git --git-dir=$HOME/.sysconfig.git --work-tree=/'
```

Use `sc` everywhere you would otherwise type `git` (`sc status`, `sc add /etc/foo.conf`, `sc commit`, `sc push`). The repo has `status.showUntrackedFiles=no` so `sc status` only reports modifications to already-tracked files — that is what replaces the old allowlist `.gitignore`. Adding a new file is just `sc add /absolute/path` with no parent-directory ceremony.

Currently tracked: `CLAUDE.md`, `etc/keyd/default.conf`, `usr/local/bin/fcitx5-toggle`.

## The two managed components

### `/etc/keyd/default.conf` — keyd remap

keyd runs as root via `keyd.service`, reads evdev events, and emits remapped events through `/dev/uinput`. The config defines four layers:

- `[main]` — physical-key bindings (LeftAlt→cmd, LeftMeta→opt, CapsLock→ctrl_layer+toggle command).
- `[ctrl_layer:C]` — empty body; `:C` suffix auto-prefixes Ctrl on every chord (CapsLock-hold replicates "CapsLock as Ctrl").
- `[cmd:C]` — Mac Cmd layer; `:C` makes letter chords auto-Ctrl. Explicit overrides cover caret motion (`left=home`, `right=end`, `up/down=C-home/C-end`).
- `[opt:A]` — Mac Option layer; `:A` auto-prefixes Alt. Explicit overrides cover word-jump (`left=C-left`, `right=C-right`, `backspace=C-backspace`).

After editing: `sudo systemctl restart keyd` (changes are not picked up automatically). Use `journalctl -u keyd --no-pager` for diagnostics; `DEVICE: match` lines confirm which keyboards keyd grabbed.

### `/usr/local/bin/fcitx5-toggle` — out-of-band IME toggle

Called by keyd's `command()` action on CapsLock-tap. keyd runs as root, so the script uses `runuser -u fenrir --` plus explicit `DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus`, `XDG_RUNTIME_DIR=/run/user/1000`, `DISPLAY=:0` to reach the user's fcitx5 daemon and call `fcitx5-remote -t`.

This deliberately bypasses the keysym chain. An earlier design used keyd→`F13`→fcitx5 `TriggerKeys=F13`, but xkb maps `KEY_F13` to `XF86Tools`, which GNOME's `media-keys` schema binds to "open Settings" — GNOME ate the event before fcitx5 saw it. The DBus path sits above the compositor, so nothing can intercept it.

Constraints:
- Must be owned `root:root` mode `0755` (keyd refuses to exec otherwise).
- The hardcoded `1000` must match `id -u fenrir`. If the UID ever changes, edit both `/run/user/1000/bus` and `XDG_RUNTIME_DIR=/run/user/1000`.
- Test directly with `/usr/local/bin/fcitx5-toggle; echo $?` — non-zero usually means fcitx5 isn't running or the runtime dir path is wrong.

## Operational notes

- **Root-shell `HOME` trap.** This Claude Code session likely runs as root; `~` expands to `/root`, not `/home/fenrir`. Always use absolute paths when writing user-side files (e.g. `/home/fenrir/.config/...`), or prefix with `runuser -u fenrir --`. The `Write` tool takes absolute paths and is safe; `Bash` with `~` is not.
- **Backups.** Before editing `/etc/keyd/default.conf`, save a timestamped sibling (`default.conf.bak.$(date +%s)`); the rollback procedure in `/docs/mac-keyboard-feel.md` §8.2 expects them.
- **Verification.** Section 7 of `/docs/mac-keyboard-feel.md` is the canonical 10-row test matrix (Cmd+T, Cmd+Left, CapsLock-tap toggling fcitx5, etc.). Run it from a freshly-launched app — already-running processes hold stale `*_IM_MODULE` env.
- **Wayland gotcha.** `~/.xprofile` is dead under Wayland; IM env vars must live in `~/.config/environment.d/*.conf` (parsed by systemd, so `KEY=VALUE` only — no `export`, no shell logic).

## The `/docs/` directory

`/docs/mac-keyboard-feel.md` is the long-form design doc covering architecture, every config file's full contents (including the user-side fcitx5 files that are **not** tracked here), setup-from-scratch, rollback, troubleshooting, and lessons learned. It is intentionally **not** tracked by git but is the authoritative reference — read it before making non-trivial changes.
