# CLAUDE.md

## Overview

PodSight shows live thumbnail previews of running EVE Online clients on
Linux, for multiboxing: click to focus a client, drag to reposition, zoom on
hover, plus global switching hotkeys, pins, snapping, saved layouts and a
tray icon. Python 3.8+ and GTK3/libwnck via PyGObject. Runs on X11, or on
Wayland through XWayland, with optional gtk-layer-shell overlays above
fullscreen clients. Version lives in `podsight_pkg/__init__.py`.

## Install, run, test

There is no `pyproject.toml` or other packaging; the app runs straight from
the repo. System dependencies (GTK3, libwnck3, PyGObject; optional wmctrl,
xdotool, gtk-layer-shell, ayatana-appindicator) are listed per distro in the
README.

```bash
python3 podsight.py                       # run from the repo
python3 podsight.py --debug               # per-frame capture diagnostics
python3 podsight.py --stats               # perf counters to stderr every 5 s
PODSIGHT_IPC_DEBUG=1 python3 podsight.py  # trace layer-shell IPC

./install.sh     # per-user install to ~/.local; creates the `podsight` command
./uninstall.sh   # removes everything install.sh created
```

The installed `podsight` launcher logs to `~/.local/state/podsight/podsight.log`.
Config is `~/.config/podsight/config.json`.

There is no automated test suite. `python3 -m py_compile podsight.py
podsight_pkg/*.py` catches syntax errors; anything else needs a manual run
against real EVE clients on the development hosts.

## Module layout

- `podsight.py`: entry point; flag docs, tray wiring, starts the GTK main loop.
- `podsight_pkg/platform.py`: session detection, sets GDK_BACKEND before gi loads, X11 ctypes helpers. Import it first in any module that touches gi.
- `podsight_pkg/app.py`: EVE client discovery (including launcher filtering) and the management window.
- `podsight_pkg/thumbnail.py`: `ThumbnailWindow`, one live preview per client, including the capture loop.
- `podsight_pkg/capture.py`: server-side scaled capture via XComposite + XRender (ctypes).
- `podsight_pkg/layer_shell.py`: Wayland layer-shell OVERLAY helper subprocess and its IPC wrapper.
- `podsight_pkg/hotkeys.py`: Ctrl+Alt client-switching hotkeys via XGrabKey.
- `podsight_pkg/tray.py`: StatusNotifier tray icon via AyatanaAppIndicator (optional).
- `podsight_pkg/settings_dialog.py`: Settings dialog.
- `podsight_pkg/config.py`: config load/save, with one-time migration from the old app name.
- `podsight_pkg/stats.py`: opt-in `--stats` performance counters.

`CODE_WALKTHROUGH.md` has the longer design write-up.

## Development hosts

- **forge**: Ubuntu 26.04, GNOME
- **hammer**: Fedora 44, KDE

## Rules

- No em-dashes in docs (README, CODE_WALKTHROUGH, this file) or user-facing
  text (UI labels and tooltips, printed messages, installer output). Code
  comments and docstrings are exempt.

## Hard-won lessons

- `Gdk.pixbuf_get_from_window()` is a synchronous bottleneck (full-window
  XGetImage over the X socket, then CPU scaling). Capture uses XComposite +
  XRender via ctypes (`podsight_pkg/capture.py`); the Gdk path remains only
  as a fallback when `grab_scaled()` returns None.
- XGetImage on pixmaps reports zero RGB masks, because masks come from a
  visual and pixmaps have none. The destination is always PictStandardRGB24,
  so channel order defaults to BGRX (little-endian x8r8g8b8). Do not derive
  it from the masks alone.
- Grid snapping needs a float accumulator. Quantizing the accumulated
  position itself rounds every small delta back to the same cell, and the
  feedback freezes the drag (see the live drag in `podsight_pkg/layer_shell.py`).
- Launcher detection (`podsight_pkg/app.py`): accept the resolved window
  title ("EVE - <Character>") before any cmdline heuristics, and use
  specific signatures such as "qtwebengine", not short substrings like
  "qml", which appear in random data such as JWTs (the ssoToken) in process
  arguments.

## Working style

- Plan each milestone with goal, files, commands and explicit pass criteria
  before coding. Debug with ranked hypotheses first.
- Edit files in place; never deliver changes as zip or patch files.
- Do not commit or push unless asked. The user reviews the diff first.
