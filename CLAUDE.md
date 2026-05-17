# Repo overview

This is a fork of [tdryer/eh-fifty](https://github.com/tdryer/eh-fifty)
(a reverse-engineered Python library for the Astro A50 Gen 4 wireless
headset). The fork hosts two things:

1. **Library** at the repo root (`eh_fifty.py`, `tests.py`, etc.) — kept
   in sync with upstream. Custom commits here are destined for an upstream
   pull request.
2. **GUI** under `gui/` — a Qt application built on the library, specific
   to this fork (not intended for upstream).

## Branches

- `master` → tracks `upstream/master` (tdryer). **Protected**: no
  force-push, no deletion. Fast-forwards only.
- `add-device-info-and-firmware-version` → upstream-bound PR adding 5
  reverse-engineered HID opcodes (device info, FW versions, FW build date)
  plus a session-scoped backup/restore pytest fixture.
- `gui` → all GUI work lives here. PR #1 targets `master` of the fork
  (not upstream).

When sending fixes / additions to the library, branch from `master` and
target `tdryer/eh-fifty:master`. When working on the GUI, branch from
`gui`.

## Library (root)

- `eh_fifty.py` — core HID protocol implementation
- `tests.py` — pytest tests for the library (run against a real A50)
- `conftest.py` — session-scoped autouse fixture that snapshots and
  restores the device configuration around the test session
- Tooling per `pyproject.toml`: hatch, black (line length 88), isort
  (black profile), pylint (`disable=fixme,too-many-public-methods`,
  `score=no`), mypy strict
- Python 3.7+ supported — use `from __future__ import annotations` if you
  add parameterised type hints to library files

### Running the library tests

The tests randomise device state, so the fixture in `conftest.py`
captures the saved/active state before the session and restores both
afterward. **Always run with a real A50 attached.**

```bash
hatch run pytest tests.py
```

### Important caveat

Never call `device._dev.reset()` (or rely on `eh_fifty._request`'s
post-timeout `dev.reset()` path) — on the A50 this disconnects the base
from USB and forces a physical replug. Wrap HID writes in code that
*never* reaches the timeout path during normal operation.

## GUI (`gui/`)

Standalone PyQt6 application — see `gui/README.md` for end-user docs.

Source layout (each module is a focused concern):

| File                  | Role                                                  |
|-----------------------|-------------------------------------------------------|
| `gui.py`              | `A50Window` (main window), wiring, `main()`           |
| `eq_widget.py`        | `EqTemplatesWidget` — full EQ preset feature (radios, combos, meter, state) |
| `eq_meter.py`         | `_EqMeter` — interactive 5-band bargraph              |
| `templates.py`        | Builtin `_EQ_TEMPLATES`, user-template JSON I/O       |
| `i18n.py`             | `TRANSLATIONS` (fr/en), `t()`, `_detect_lang()`       |
| `raw_request.py`      | Raw HID opcodes outside eh-fifty's public API         |
| `process_lock.py`     | Single-instance helper (`/proc` scan)                 |
| `status_worker.py`    | `QObject` polling status on a `QThread`               |
| `base_info_dialog.py` | Formats the "Informations base" dialog content        |
| `menu_install.py`     | KDE menu entry install / remove (.desktop file)       |

### Threading

The A50 device handle is shared between the main UI thread and the
`StatusWorker` thread. A `threading.RLock` (`A50Window._device_lock`)
serialises every USB access. **Whenever you add code in the main thread
that calls `self.device.*` outside a single trivial call, wrap the burst
in `with self._device_lock:` so it doesn't race the worker's poll.**

### Tests

`gui/tests.py` contains 14 unit tests using the standard library's
`unittest` (no pytest dependency). They cover pure logic — builtin
template shape, user-template JSON round-trip, BCD/datetime helpers,
i18n lookups and FR↔EN key parity. They do **not** require a real
device.

```bash
cd gui && /path/to/.venv/bin/python -m unittest tests.py
```

### User-defined preset storage

User EQ presets are persisted to
`$XDG_CONFIG_HOME/astro-a50-gui/user-templates.json` (or
`~/.config/astro-a50-gui/...` if `XDG_CONFIG_HOME` is unset). The 5
builtin templates (A50 MOD KIT, ASTRO, MEDIA, PRO, STUDIO) are immutable
constants in `templates.py` — never persist them to the user file.

## A50 protocol notes

### USB IDs

- **`9886:002c`** → A50 Gen 4 (PS4/PC variant, switch on back), normal mode
- **`9886:002a`** → same hardware, **bootloader** ("Polaris") — appears
  briefly during firmware update, then re-enumerates back to `002c`
- `9886:0015` → Gen 3 (not supported by this fork)

### HID probing pitfalls

If you reverse-engineer new opcodes by sending raw requests, keep these
in mind:

- **Never call `dev.reset()` after a timeout.** eh-fifty's `_request()`
  does this internally on timeout, which on the A50 disconnects the
  base from the USB bus and requires a physical replug to recover. If
  you probe via raw HID, patch out the reset path or wrap calls so the
  timeout case never fires (e.g. write your own request that catches
  `usb.core.USBError` without resetting).
- Opcode `0x08` returns `HID_ERROR_I2C_BUS_ERROR` immediately.
- Opcodes `0x0A` and `0x59` hang the firmware (timeout). Combined with
  the reset pitfall above, hitting these is a hard recovery scenario.
- Most undefined opcodes return `HID_ERROR_UNHANDLED_PACKET`.

### Orphan opcodes on Gen 4

`0x71 SET_MIC_EQ` / `0x7B GET_MIC_EQ` exist in the firmware but appear
to be dead code on Gen 4 — Astro Command Center never invokes them, and
a live loopback test on this hardware (mic → game sink while cycling
presets 0/1/2) produced no audible difference. Treat as unsupported on
Gen 4; the GUI deliberately doesn't expose a Mic EQ control. The
docstring on `get_mic_eq` even says "Their meanings are unknown".

### Capturing ACC traffic

To extend the protocol, capture USB traffic from Astro Command Center
running in a Windows VM (libvirt USB passthrough). usbmon on the host
captures the traffic at the xhci layer, *before* QEMU forwards it, so
the capture works even while the device is passed through to the guest.

The fork ships two helpers under `gui/`:

- `hid-capture.sh` — wraps `usbmon` to record HID traffic to a text file
- `hid-parse.py` — pairs OUT/IN URBs into request/response, decodes
  payload bytes, and tries to map opcodes back onto `eh_fifty._CommandType`

### Noise gate mode ↔ ACC FR labels

When matching captured opcodes against the ACC UI in French:

| `NoiseGateMode` | ACC FR     |
|-----------------|------------|
| `STREAMING` (0) | Diffusion  |
| `NIGHT` (1)     | Nuit       |
| `HOME` (2)      | Maison     |
| `TOURNAMENT` (3)| Tournoi    |

The ACC "Mic" panel in French is the **noise gate**, not Mic EQ.
