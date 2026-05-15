# wled binary spec

## api info

current wled instance is on `192.168.100.185`
`/json` -> `state`, `info`, `effects`, `palettes`
`/json/state` -> only `state` <- THE ONE WE WANT, [read-write]
`/json/info` -> only `info` <- [read-only]

sample `state`:

```json
{
  "on": false,
  "bri": 255,
  "transition": 7,
  "bs": 0,
  "ps": -1,
  "pl": -1,
  "ledmap": 0,
  "AudioReactive": {
    "on": false
  },
  "nl": {
    "on": false,
    "dur": 60,
    "mode": 1,
    "tbri": 0,
    "rem": -1
  },
  "udpn": {
    "send": false,
    "recv": true,
    "sgrp": 1,
    "rgrp": 1
  },
  "lor": 0,
  "mainseg": 0,
  "seg": [
    {
      "id": 0,
      "start": 0,
      "stop": 120,
      "len": 120,
      "grp": 1,
      "spc": 0,
      "of": 0,
      "on": true,
      "frz": false,
      "bri": 255,
      "cct": 127,
      "set": 0,
      "col": [
        [243, 207, 160],
        [0, 0, 0],
        [0, 0, 0]
      ],
      "fx": 0,
      "sx": 128,
      "ix": 128,
      "pal": 50,
      "c1": 128,
      "c2": 128,
      "c3": 16,
      "sel": true,
      "rev": false,
      "mi": false,
      "o1": false,
      "o2": false,
      "o3": false,
      "si": 0,
      "m12": 0
    }
  ]
}
```

`POST /json/state` will update the current state
note: the object doesnt have to be full for state to update
eg. `POST /json/state` with `{ "on": true, "bri": 255}` will turn the wled on and set it to max brightness

`{"on": "t"}` works as a toggle

`{ "bri": [range] }` — range should only be [1, 255]

## features

- turn on/turn off/toggle state of wled
- should be able to run it as binary from any dir in the terminal, will use `#!/usr/bin/env python3` shebang for that
- wont have `.py` termination so it's only `wled` when called
- wont use thrid-party libraries, only what's available in the standard library such as `import json`
- is able to change ip address of the instance, for persistance, save it in a file named `config.json` with value `"ip": "<ip addr>"`, will probably need this file for later, more advanced configuration of the program
- at first run it prompts you to run `wled ip <value>` to set an ip address to use and then check to see if it's a correct and available address by requesting `GET /json/info` and saving it in `info.json`
- will use that `info.json` file for inspecting current set instance
- when using `wled state | wled status` it should save the last state in `last-state.json` as well so when i check state it reads from this unless a variable `STATE_CHANGED == true` which only changes when i make changes to the state and on next state request if it's true it requests the wled instance its current state instead of reading from file, updates the local `last-state.json` and finally updates the variable `STATE_CHANGED = false` — should add this in `config.json` because wled is stateless, it doesnt run continuosly
- ability to scan local network for wled instances, and saves them in `instances.json`, and then being able to select one of them with `wled select`. the command by itself will prompt you to choose an instance either by ip, id (generated for each new instance in `instances.json`) or name

## usage

- `wled` — will toggle the instance with `{ "on": "t" }`
- `wled on` — will turn on
- `wled off` — will turn off
- `wled state | wled status` — will return to the screen a nice interface that shows current state (stub for now because we only work with on/off and brightness)
- `wled 127` — will set brightness value IF wled is on, if off, will return the message `turn it on first!`
- `wled max` — will set `wled 255` essentially, but it will go through the same flow, if wled state off it will again say `turn it on first!`
- `wled on 127` — will turn it on first and then set brightness to that value
- `wled help | wled -h | wled --help` — will show a typical help menu for the binary
- `wled ip <value>` — set the ip address of the instance, will persist between runs
- `wled info` — reads from `info.json` and outputs relevant info in terminal
- `wled scan` — checks current network for wled instances and overwrites `instances.json` with a list of found instances; an instance looks like `{ "id": id (int), "name": "name" (str), "ip": "ip" (str) }`
- `wled select` — will show a list of available instances only from `instances.json`, will select the first one by highlighting it and changing the color to the terminal primary color and a right arrow to the lft of it, will be able to move between highlighted instances with up and down arrow, and enter will select it. also ability to manually type in the id, name or ip and enter to select. will show above currently selected instance if any
- `wled select <instance>` — selects an instance from `instances.json`, `<instance>` can be id, name or ip
- `wled config` — where you customize settings related to the selected instance, enters a command line interface with `wled[name_of_instance|ip_addr] >` and above it graphics that show state and name, segments with ability to make more segments visually — DO NOT IMPLEMENT YET, HAVE TO ARCHITECT THIS MORE, THIS IS PLANNED FOR THE FUTURE

## interface/code snippets

part code, part implementation observations

### state

the state of the wled instance

#### display

for the `wled state | wled status` it will look something like this

```txt
WLED State (192.168.100.185)
═══════════════════════════
Power   : ON
Brightness : 68% [██████████░░░░░░░░░░] (173/255)
Effect  : Solid (0)
Palette : Default (50)
Segment : 0 (0‑120 LEDs) ON
```

and for that bar we can use this python function

```py
def brightness_bar(bri, width=20):
    filled = int(bri / 255 * width)
    empty = width - filled
    return '█' * filled + '░' * empty
```

- use █ (U+2588) for filled, ░ (U+2591) for empty

#### state change

```py
STATE_CHANGED = False   # persisted in a tiny file or just in memory; reset on each run
def state_command():
    if not STATE_CHANGED:
        data = load_json('last-state.json')
    else:
        data = fetch_from_wled('/json/state')
        save_json('last-state.json', data)
        STATE_CHANGED = False
    display(data)
def post_command(payload):
    fetch_from_wled('/json/state', method='POST', data=payload)
    STATE_CHANGED = True   # next state call will refresh from the device
```

- will have to have a `update_config()` at the end of the program to update variables like `STATE_CHANGED` in the file

### libraries

i want to keep it self contained and lean in terms of dependencies, and by that i mean no external dependencies, at all

- `json` for parsing json, its a standard module
- will go with `urllib.request`

### scanning

will be using subnet scan with only /24 for now (that covers most use cases)
multi threaded search for `/json/info` on every available host

something like this

```py
import ipaddress, threading, urllib.request, json

  subnet = "192.168.100.0/24"
  found = []

  def probe(ip):
      try:
          with urllib.request.urlopen(f"http://{ip}/json/info", timeout=0.5) as r:
              info = json.load(r)
              if "leds" in info:  # WLED-specific field
                  found.append((ip, info.get("name")))
      except:
          pass

  threads = [threading.Thread(target=probe, args=(str(h),))
             for h in ipaddress.ip_network(subnet).hosts()]
  for t in threads: t.start()
  for t in threads: t.join()
  print(found)
```

should also show a progress bar and information about where we're at in the scanning process and add found instances above the progress bar with their ip and name
