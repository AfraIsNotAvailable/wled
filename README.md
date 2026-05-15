# wled

Command-line controller for [WLED](https://kno.wled.ge) instances. No dependencies — standard library only.

## Installation

```bash
chmod +x wled
sudo cp wled /usr/local/bin/
```

Or add the repo directory to your `$PATH`.

## Setup

On first use, point it at your WLED instance:

```bash
wled ip 192.168.1.100
```

Validates the connection, saves device info to `info.json`.

## Usage

```
wled                toggle power
wled on             turn on
wled on <1-255>     turn on at brightness
wled off            turn off
wled <1-255>        set brightness (device must be on)
wled max            set brightness to 255 (device must be on)
wled state          show current state
wled status         alias for state
wled ip <addr>      set WLED IP address
wled info           show saved device info
wled help           show this help
wled -h, --help     show this help
```

## State display

```
WLED State (192.168.1.100)
═════════════════════════════
Power      : ON
Brightness : 68% [█████████████░░░░░░░] (173/255)
Effect     : (0)
Palette    : (50)
Segment    : 0 (0-120 LEDs) ON
```

## Local files

| File | Purpose |
|------|---------|
| `config.json` | Persisted IP and state-changed flag |
| `info.json` | Device info snapshot (written on `wled ip`) |
| `last-state.json` | Cached state (refreshed only when state changes) |

These are created next to the binary and are excluded from version control.

## Spec

See [`wled-spec.md`](wled-spec.md) for the full feature spec and API notes.
