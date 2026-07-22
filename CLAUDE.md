# kegboard — Claude Code context

## What this is
Fork of [Kegbot/kegboard](https://github.com/Kegbot/kegboard) (origin: `flangelo/kegboard`).
Contains the Arduino firmware, hardware designs, docs, and a Python support
library for the Kegboard flow-meter controller. Upstream has been dormant since
2017; local changes sit on top of it.

The physical board in production is an Arduino Uno attached to the Pi
(`frode@kegberry`) at `/dev/ttyACM0`, running **firmware v18**. It is read by
the kegboard serial daemon from **kegbot-pycore** (container
`kegberry-kegboard-1`), not by anything in this repo.

## Layout
```
arduino/kegboard/   — firmware sketch (kegboard.ino) + libs; version.h holds FIRMWARE_VERSION
hw/                 — board designs (kegboard-mini, kegboard-coaster, kegboard-mega)
python/kegbot/kegboard/ — Python serial-protocol library (see the big caveat below)
docs/source/        — Sphinx docs; serial-protocol.rst is the KBSP protocol reference,
                      firmware.rst covers building/flashing
```

## ⚠️ The python/ library here is NOT what runs in production
kegbot-pycore installs `kegbot-kegboard = "*"` from **PyPI** (currently 1.2.0,
locked in its `Pipfile.lock`). The `python/` tree in this repo is the old
Python-2-era source and is **not** packaged, installed, or deployed anywhere.

Consequence, verified 2026-07-22: commit `4173236` (June 2026, "Fix serial
message robustness") patched CRC validation and `KBSP_MAXLEN` in
`python/kegbot/kegboard/`, but the deployed library inside
`kegberry-kegboard-1` still computes the CRC and never compares it — the fix
never shipped. To change production serial parsing, patch it in the PyPI
`kegbot-kegboard` package as consumed by kegbot-pycore (vendor it into pycore
or install from a git URL in pycore's Pipfile), not here.

The firmware half of that same commit (atomic `gMeters[]` reads, CRC check on
incoming commands) only takes effect when the board is reflashed; it does not
bump `FIRMWARE_VERSION`, so a `HelloMessage` reporting `firmware_version=18`
does not tell you whether it's flashed. As of 2026-07-22 there is no evidence
it ever was (no avr toolchain on the Pi, no flash history).

## Firmware build/flash
`arduino/kegboard/Makefile` is edam's general-purpose Arduino makefile:
`make` builds, `make upload` flashes via avrdude (needs `ARDUINODIR` pointing
at an Arduino IDE install; `SERIALDEV` is auto-guessed). `docs/source/firmware.rst`
describes the Arduino-IDE route instead.

Flashing notes for the production board:
- Neither the Pi nor this Mac has avrdude/Arduino tooling installed today —
  it must be installed (Pi: `apt install arduino-core avrdude` or use arduino-cli)
  before `make upload` will work.
- Stop the serial daemon first (`ssh kegberry "docker stop kegberry-kegboard-1"`)
  so `/dev/ttyACM0` is free, flash, then `docker start` it.
- Bump `FIRMWARE_VERSION` in `arduino/kegboard/version.h` with any behavior
  change, so the running version is observable from the `HelloMessage`.

## Runtime behavior / debugging (production board)
- Firmware sends `HelloMessage` every 10 s (with uptime), temperature readings
  every ~2 s per DS18B20 sensor, and `MeterStatusMessage` on flow ticks
  (`flow0`/`flow1`). Meter counts live in RAM — they zero on board reset.
- Watch it: `ssh kegberry "docker logs -f kegberry-kegboard-1"`. Container log
  timestamps are **UTC**; the Pi is Pacific time.
- The serial link occasionally wedges silent; the pycore daemon's watchdog
  reconnects after 30 s of silence, and reopening the port toggles DTR, which
  **resets the Arduino** (uptime and meter counters restart near zero, no USB
  re-enumeration in dmesg). An unexplained "board reset" is usually that
  watchdog recovering a wedge — check the daemon log for
  `Watchdog: no data from /dev/ttyACM0`.
- Board identity on the Pi's USB bus: Arduino Uno (2341:0043),
  serial `9543731333435181E1E2`, cdc_acm → `/dev/ttyACM0`.
