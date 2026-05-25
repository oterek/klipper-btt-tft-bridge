# klipper-btt-tft-bridge

A **Klipper `extras` module** that runs a **BigTreeTech TFT** (TFT35 E3 V3 and friends) in full **Touch Mode** against Klipper — **including host‑initiated print progress**: prints started from Mainsail/Fluidd show up on the TFT with a live progress bar.

It fixes the two problems everyone hits when wiring a BTT TFT to a Klipper machine:

1. **`ACK timed out` / `pending gcode released`** spam (especially during prints).
2. **No print status on the TFT** when the print is started from the host (Mainsail/Fluidd).

> As far as we could find, no public solution covered the full Touch‑Mode experience **with host print progress** — most guides stop at "use the 12864 emulation menu" or "use KlipperScreen instead". This does the real thing.

---

## How it's different from existing bridges

Transparent bridges (e.g. `oldhui-uk/tftbridge`, plain `socat` to `/tmp/printer`) just shuttle bytes between the TFT and Klipper. That breaks because:

- The BTT TFT speaks **Marlin** and expects Marlin‑shaped replies and timing.
- A naive bridge often sends an extra `ok` **and** lets Klipper send its own → **double `ok` → ACK desync → `ACK timed out`**.
- Klipper rejects Marlin‑only commands (`M92`, `M211`, `M420`, …) with `!! Unknown command`, which the TFT treats as an error.
- Long commands (`G28`, heating, bed mesh) don't return `ok` for many seconds → the TFT's hard ~5 s ACK timeout fires.
- Nothing ever tells the TFT that a **host** print is running, so it never shows progress.

This module is a **Marlin‑compatibility layer** that runs *inside* Klippy and answers the TFT the way it expects.

---

## Features

- **In‑process** Klipper `extras` module — no extra daemon, no `socat`, no `/tmp/printer` pty round‑trip.
- **Single‑threaded** via `reactor.register_fd` (the same mechanism Klipper uses for its own serial) — no thread races.
- **Local, instant status replies** straight from Klipper's object model: `M105` (temps), `M114` (position), `M220`/`M221` (feed/flow), `M27` (SD status). These never enter the gcode queue, so the status loop no longer fights print/motion traffic.
- **Exactly one `ok` per command** — kills the double‑ack that causes `ACK timed out`.
- **`echo:busy: processing` keepalive** during long commands (`G28`, `M109/M190`, bed mesh) so the TFT never times out waiting.
- **`M155` temperature auto‑report** — temps keep updating even while a long command runs.
- **`M115`** advertises a Marlin firmware name and the capabilities the TFT needs (`AUTOREPORT_TEMP:1`, `SDCARD:1`, `AUTOREPORT_SD_STATUS:0`).
- **Host print progress** — detects a print start from `print_stats`, tells the TFT (`File opened: …`), which makes the TFT open its print screen and poll `M27` (answered with live `SD printing byte X/Y`).
- **Marlin → Klipper translations**: `G29`→`BED_MESH_CALIBRATE`, `M420 S1`→`BED_MESH_PROFILE LOAD=default` / else `BED_MESH_CLEAR`, `M851 Z`→`SET_GCODE_OFFSET`, `M24`/`M25`/`M524`→`RESUME`/`PAUSE`/`CANCEL_PRINT`.
- **Garbage‑byte tolerant** — the TFT occasionally injects junk bytes (e.g. `0xFD`) before a command; the bridge decodes with `errors='ignore'` and never forwards raw junk into Klipper (a naive bridge crashes on the UTF‑8 decode here).

---

## Topology / wiring

```
BTT TFT (Touch Mode)  --UART/TTL-->  USB-TTL adapter (CH340/CP2102/FTDI)  --USB-->  Raspberry Pi (Klipper + Moonraker)
SKR / mainboard       ------------------------------------------------------USB-->  Raspberry Pi
```

The TFT is **not** wired to the mainboard's TFT port. It's a plain serial device on the Pi, e.g. `/dev/serial/by-id/usb-1a86_USB_Serial-...`.

> ⚠️ The BTT TFT "RS232" connector is **TTL 3.3 V**, not real RS‑232. Use a plain USB‑TTL adapter. Do **not** put a real RS‑232 (±12 V, e.g. MAX3232) converter in the path — it will garble or destroy the signal.

---

## Install

### 1. Klipper module
Copy `tftbridge.py` to your Klipper extras folder:
```bash
cp tftbridge.py ~/klipper/klippy/extras/
```

### 2. `printer.cfg`
```ini
[tftbridge]
tft_device: /dev/serial/by-id/usb-XXXXXXXX   # your USB-TTL adapter (ls /dev/serial/by-id/)
tft_baud: 115200
# optional:
# busy_interval: 2.0      # seconds between "busy: processing" keepalives
# machine_type: Ender-3   # reported in M115
```
You also need the standard Mainsail/Fluidd objects (usually already present):
```ini
[virtual_sdcard]
[print_stats]
[display_status]
[pause_resume]
```

### 3. TFT `config.ini` (on the TFT's microSD)
Set these keys, put the file on the SD card root, boot the TFT once to apply (it renames `config.ini` → `config.ini.CUR`):
```ini
serial_port:P1:6        # 115200 — must match tft_baud
command_checksum:0
onboard_sd:1            # REQUIRED for print progress (see below)
M27_always_active:1
M27_refresh_time:2
prog_source:0           # file-position based progress %
```

### 4. Apply
```bash
sudo systemctl restart klipper
```
Then **power‑cycle the TFT** so it re‑reads `config.ini` and re‑runs its `M115` handshake.

---

## Keeping Klipper's git repo clean (avoid the `-dirty` flag)

Copying `tftbridge.py` into `klippy/extras/` adds an **untracked file** to Klipper's git repo, so Moonraker reports the Klipper version as `…-dirty` and its update manager may refuse to auto-update Klipper. Tell git to ignore the module locally:

```bash
echo 'klippy/extras/tftbridge.py' >> ~/klipper/.git/info/exclude
```

Also keep any backup copies (`tftbridge.py.bak`, …) **outside** `klippy/extras/` — leftover files there dirty the repo the same way.

---

## The part nobody documents: host print progress

The TFT only polls `M27` once it *believes a print is active* — and `M27_always_active` does **not** make it poll while idle (despite the name; it means "also poll for prints not started from the TFT").

So the bridge has to actively kick the TFT into print mode. From the firmware source (`Mainboard_AckHandler.c`):

- An unsolicited line `File opened: <name> Size: <bytes>` triggers `startPrintingFromRemoteHost()` — **gated on `onboardSD == ENABLED`** (hence `onboard_sd:1` is mandatory). The TFT opens its print screen and calls `request_M27()`.
- From then on the TFT polls `M27`, which the bridge answers with `SD printing byte <pos>/<size>` (and `Done printing file` on completion).

The bridge watches `print_stats.state` on a 1 s timer and emits `File opened:` on the `→ printing` transition. That's the whole trick.

---

## Command handling

| From TFT | Bridge does |
|---|---|
| `M105` | local temps: `ok T:.. /.. B:.. /.. @:0 B@:0` |
| `M114` | local position from `gcode_move` |
| `M220` / `M221` (no `S`) | local current feed/flow factor |
| `M220 S` / `M221 S` | forwarded (Klipper supports these natively) |
| `M27` | local SD/print status |
| `M115` | local Marlin capability report |
| `M155 S<n>` | starts/stops local temperature auto‑report |
| `M92`, `M211`, `M500`‑`M503` | quietly acked (no‑op) |
| `G29` | → `BED_MESH_CALIBRATE` |
| `M420 S1` / else | → `BED_MESH_PROFILE LOAD=default` / `BED_MESH_CLEAR` |
| `M851 Z<v>` | → `SET_GCODE_OFFSET Z=<v> MOVE=0` |
| `M24` / `M25` / `M524` | → `RESUME` / `PAUSE` / `CANCEL_PRINT` |
| everything else | `gcode.run_script(...)`, then one `ok`; long commands get `busy: processing` keepalives |

---

## Known limitations

- **Klipper host restart** (`systemctl restart klipper`, or saving the config in Mainsail) drops the serial for ~9 s, which is longer than the TFT's hard ~5 s ACK timeout → the TFT shows a modal `ACK timed out` that you must dismiss with **OK**. Restarting just the **printer/MCU** is fine (the klippy host process stays up, the bridge keeps the port). A fully restart‑proof version would run the bridge as a separate process holding the port independently of Klipper — contributions welcome.
- The TFT's on‑board SD **file browser** is empty — the files live in Klipper's `virtual_sdcard`. Start prints from Mainsail/Fluidd; the TFT mirrors status and gives pause/resume/cancel.

---

## Tested on

- Ender 3 Pro · SKR Mini E3 V3 · Klipper `v0.13` · Raspberry Pi Zero 2 W
- BTT **TFT35 E3 V3.0 (GD)** in Touch Mode · CH340 USB‑TTL @ 115200

Other BTT TFTs (TFT24/28/35/43/50, V3) in Touch Mode should work the same way — please report results.

---

## Credits

Rewrite of the transparent‑bridge idea from [`oldhui-uk/tftbridge`](https://github.com/oldhui-uk/tftbridge), turned into a full Marlin‑compatibility layer. TFT firmware behaviour reverse‑engineered from [`BIGTREETECH-TouchScreenFirmware`](https://github.com/bigtreetech/BIGTREETECH-TouchScreenFirmware) source (`Printing.c`, `Mainboard_AckHandler.c`).

## License

MIT — see [LICENSE](LICENSE).
