# Heltec WiFi LoRa 32 V4 R8 — MQTT Observer Firmware

Build and flash MQTT observer firmware for the Heltec WiFi LoRa 32 V4 R8, from a
completely fresh machine through to a running, publishing node.

Covers **macOS and Linux**. Windows is not covered — if you need it, open an issue
and ask.

---

## ⚠️ Read this first

**These builds are experimental and AI-generated.**

- The environments described here were **generated with AI assistance** (Claude).
  They have been build-tested and hardware-tested, but they have **not** been
  through upstream review.
- Firmware built from these environments reports its version with an **`-exp`
  suffix** (for example `v1.17.1-exp`). That marker exists so an experimental
  build is never mistaken for an official release. **If your node reports a
  version ending in `-exp`, you are running unofficial firmware.**
- This is **not official MeshCore firmware** and is not supported by the MeshCore
  project or by Heltec.

**THIS SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OR GUARANTEE OF ANY KIND,
EXPRESS OR IMPLIED. ALL USE IS ENTIRELY AT YOUR OWN RISK.**

No one involved accepts liability for any damage, data loss, bricked hardware,
network disruption, missed messages, or any other loss arising from use of this
firmware. You are responsible for what you flash onto your own hardware and for
what your node transmits onto shared mesh and MQTT infrastructure.

Specific risks worth understanding before you start:

- **Flashing replaces your existing firmware.** Have a way back (see
  [Returning to official firmware](#returning-to-official-firmware)).
- **Radio transmission is regulated.** You are responsible for using a LoRa
  frequency and duty cycle legal in your country. The defaults may not be legal
  where you are.
- **An observer publishes data about your mesh** — packet metadata, signal
  readings, and your node's identity — to third-party MQTT brokers. Understand
  what you are publishing and to whom before enabling it.
- Only the **headless room server** has been tested on physical hardware. The
  OLED and TFT variants compile but have never been run on a real display.

The underlying MeshCore project is MIT licensed (see `license.txt`). That license
includes the same disclaimer of warranty and liability.

---

## What this adds

Six PlatformIO environments for the V4 R8, covering three display tiers and two
node roles.

| Environment | Display | Role |
|---|---|---|
| `heltec_v4_r8_repeater_observer_mqtt` | none (headless) | Repeater |
| `heltec_v4_r8_room_server_observer_mqtt` | none (headless) | Room server |
| `heltec_v4_r8_oled_repeater_observer_mqtt` | SSD1306 OLED | Repeater |
| `heltec_v4_r8_oled_room_server_observer_mqtt` | SSD1306 OLED | Room server |
| `heltec_v4_r8_tft_repeater_observer_mqtt` | ST7789 TFT | Repeater |
| `heltec_v4_r8_tft_room_server_observer_mqtt` | ST7789 TFT | Room server |

**Pick the one matching your board and the role you want.** If your R8 has no
screen, use the plain (no infix) name. Getting this wrong is harmless — reflash
with the right one.

Beyond the stock firmware these include the MQTT bridge, the web config portal,
an SNMP agent, and TLS via an embedded certificate bundle. The CPU runs at
160 MHz rather than the stock 80 MHz to give WiFi and TLS headroom.

---

## Hardware you need

- A **Heltec WiFi LoRa 32 V4 R8** (ESP32-S3, 16 MB flash, 8 MB PSRAM). This is
  *not* the same as a plain V4 — the R8 is a distinct board with a different pin
  map. Firmware for one will not work correctly on the other.
- A **USB-C data cable**. Charge-only cables are the single most common cause of
  "my board isn't detected". If the port never appears, try another cable first.
- An **antenna attached before powering on**. Transmitting without one can damage
  the radio.

The R8 exposes the ESP32-S3's built-in USB Serial/JTAG interface (USB ID
`303A:1001`). No separate USB-serial driver is needed on macOS or Linux.

---

## Step 1 — Install prerequisites

You need **git**, **Python 3**, and **PlatformIO Core**.

### macOS

Install the Xcode command line tools, which provide git:

```bash
xcode-select --install
```

Check Python (macOS ships Python 3; anything 3.6+ works, 3.9+ recommended):

```bash
python3 --version
```

If it's missing or too old, install via [Homebrew](https://brew.sh):

```bash
brew install python3
```

### Linux

Debian / Ubuntu / Raspberry Pi OS:

```bash
sudo apt update && sudo apt install -y git python3 python3-venv python3-pip
```

Fedora:

```bash
sudo dnf install -y git python3 python3-virtualenv
```

Arch:

```bash
sudo pacman -S --needed git python
```

### Install PlatformIO Core (both platforms)

Install into an isolated virtual environment. This keeps it away from your system
Python and makes removal trivial:

```bash
python3 -m venv ~/.platformio-venv && ~/.platformio-venv/bin/pip install platformio
```

Verify:

```bash
~/.platformio-venv/bin/pio --version
```

You should see `PlatformIO Core, version 6.x`.

Every `pio` command below uses the full path `~/.platformio-venv/bin/pio`. To type
just `pio` instead, add it to your `PATH`:

```bash
echo 'export PATH="$HOME/.platformio-venv/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

Use `~/.bashrc` instead of `~/.zshrc` if you're on bash (most Linux distributions).

> PlatformIO downloads the ESP32 toolchain (several hundred MB) on your first
> build. That happens once and is cached in `~/.platformio`.

---

## Step 2 — Linux only: serial port permissions

On Linux, your user needs permission to access the serial port. Without this,
flashing fails with a permission error.

```bash
sudo usermod -a -G dialout $USER
```

On Arch and some other distributions the group is `uucp` instead:

```bash
sudo usermod -a -G uucp $USER
```

**You must log out and back in** for this to take effect. Verify with:

```bash
groups | grep -E 'dialout|uucp'
```

macOS needs no equivalent step.

---

## Step 3 — Get the source

```bash
git clone https://github.com/tgodfrey/MeshCore.git
cd MeshCore
git checkout feat/heltec-v4-r8-observer-mqtt-prefs
```

Confirm you're on the right branch:

```bash
git branch --show-current
```

---

## Step 4 — Find your board's serial port

Plug the board into USB, then:

**macOS:**

```bash
ls /dev/cu.usbmodem*
```

Typically `/dev/cu.usbmodem1101`. Use the `cu.` device, not `tty.`.

**Linux:**

```bash
ls /dev/ttyACM*
```

Typically `/dev/ttyACM0`.

Either platform, PlatformIO can also list ports with hardware IDs — look for
`303A:1001`:

```bash
~/.platformio-venv/bin/pio device list
```

If nothing appears: try a different USB cable (see
[Hardware you need](#hardware-you-need)), then a different port.

---

## Step 5 — Configure the upload port

Create `platformio.local.ini` in the repository root. This file is gitignored, so
your local settings never end up in a commit:

```ini
[env:heltec_v4_r8_room_server_observer_mqtt]
upload_port = /dev/cu.usbmodem1101
monitor_port = /dev/cu.usbmodem1101
board_upload.use_1200bps_touch = no
board_upload.wait_for_upload_port = no
```

Replace the port with yours from Step 4, and the environment name with whichever
one you chose from the table above.

> **Why disable the 1200 bps touch?** That reset trick is for boards with a
> separate USB-serial chip. The R8 uses the ESP32-S3's built-in USB Serial/JTAG,
> where the touch causes the port to disconnect and re-enumerate mid-upload,
> producing a confusing `the port doesn't exist` error.

---

## Step 6 — Build

Substitute your chosen environment name throughout:

```bash
~/.platformio-venv/bin/pio run -e heltec_v4_r8_room_server_observer_mqtt
```

The first build takes several minutes while the toolchain downloads. Later builds
take well under a minute.

Success looks like:

```
RAM:   [==        ]  21.6% (used 70704 bytes from 327680 bytes)
Flash: [===       ]  25.0% (used 1640133 bytes from 6553600 bytes)
========================= [SUCCESS] Took 43.97 seconds =========================
```

---

## Step 7 — Flash

**This overwrites the firmware currently on your board.** Read
[Returning to official firmware](#returning-to-official-firmware) first if you
might want to go back.

Close anything else using the serial port — this is the most common flashing
failure. Web flashers and browser-based MeshCore tools hold the port open through
Web Serial even when idle. **Disconnect them in the browser, or close the tab.**

```bash
~/.platformio-venv/bin/pio run -e heltec_v4_r8_room_server_observer_mqtt -t upload
```

Success ends with:

```
Wrote 1640133 bytes ... 
Hash of data verified.
Leaving...
Hard resetting via RTS pin...
========================= [SUCCESS] =========================
```

If it fails, see [Troubleshooting](#troubleshooting).

---

## Step 8 — Connect to the console

```bash
~/.platformio-venv/bin/pio device monitor -p /dev/cu.usbmodem1101 -b 115200
```

Exit the monitor with **Ctrl+C**.

Press Enter to get a prompt, then confirm what you're running:

```
ver
```

Expect something like `-> v1.17.1-exp (Build: 14 Aug 2026)`. **The `-exp` suffix
confirms this is the experimental firmware.**

Check the role matches what you flashed:

```
get role
```

> There is no `help` command. The observer-specific commands (WiFi, MQTT slots,
> webconfig) are documented in
> [`MQTT_IMPLEMENTATION.md`](MQTT_IMPLEMENTATION.md); general MeshCore commands
> are in [`docs/cli_commands.md`](docs/cli_commands.md).

---

## Step 9 — Configure WiFi

The observer needs internet access to reach MQTT brokers.

```
set wifi.ssid YourNetworkName
set wifi.pwd YourPassword
reboot
```

After rebooting, watch the console — you should see the WiFi connect and the MQTT
bridge start.

---

## Step 10 — Configure MQTT brokers

Brokers live in numbered **slots** (`mqtt1`, `mqtt2`, `mqtt3`). Each slot is
either a named **preset** or a **custom** broker.

List the available presets (the output is paginated; the trailing `next:14` tells
you where to resume):

```
get mqtt.presets
```

Assign a preset to a slot:

```
set mqtt1.preset meshmapper
```

A preset can only occupy one slot — assigning a duplicate returns
`Error: preset 'x' is already assigned to slot N`. That is expected, not a fault.

For a broker with no preset, use `custom` and fill in the details:

```
set mqtt2.preset custom
set mqtt2.server mqtt.example.org
set mqtt2.port 1883
set mqtt2.username myuser
set mqtt2.password mypassword
```

Available per-slot settings: `preset`, `server`, `port`, `username`, `password`,
`topic`, `token`, `audience`.

Identify your node to the analyzers you publish to:

```
set mqtt.origin MyNodeName
set mqtt.owner YourName
set mqtt.email you@example.com
set mqtt.iata SEA
```

Save and restart:

```
save
reboot
```

---

## Step 11 — The web config portal

Easier than the serial console for ongoing changes. With WiFi connected:

```
start webconfig
```

The reply gives you a URL, for example
`WebConfig started: http://192.168.7.40/ (admin password login)`. Open it from any
device on the same network and log in with the admin password (default:
`password` — **change it**).

Shut it down when finished:

```
stop webconfig
```

Run `start webconfig ap` to force the board to raise its own access point instead
— useful before WiFi is configured.

> Leaving the portal running exposes a configuration interface on your LAN. Stop
> it when you aren't using it.

---

## Step 12 — Verify it's working

```
get mqtt.status
```

A healthy node shows each configured slot connected:

```
-> msgs: on, 1: analyzer-us (ok), 2: meshmapper (ok), 3: custom (ok), q:0
```

`q:0` is the outbound queue depth — a number that climbs and never falls means
messages aren't being delivered.

In the console you should also periodically see:

```
MQTT: Status published successfully, next publish in 300000 ms
```

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Could not open <port>, the port doesn't exist` | Usually the port is **busy**, not missing. Close browser tabs using Web Serial, and any open serial monitor. On macOS check with `lsof /dev/cu.usbmodem1101`. |
| Port vanishes during upload | The 1200 bps touch. Confirm `board_upload.use_1200bps_touch = no` is set (Step 5). |
| No port appears at all | Charge-only USB cable (most likely), dead port, or board not powered. Try another cable first. |
| `Permission denied` on the port (Linux) | Not in the `dialout`/`uucp` group, or you haven't logged out and back in since adding yourself (Step 2). |
| `fatal error: Timezone.h: No such file` | You're on the wrong branch. Confirm with `git branch --show-current`. |
| Build fails right after cloning | Interrupted toolchain download. Retry; if it persists, `rm -rf ~/.platformio` and rebuild. |
| `Unknown command` in the console | That command doesn't exist in this build. There is no `help`; see [`MQTT_IMPLEMENTATION.md`](MQTT_IMPLEMENTATION.md). |
| Board boots but never joins WiFi | Wrong SSID/password, or a 5 GHz-only network — the ESP32-S3 is **2.4 GHz only**. |
| MQTT slot stuck not connected | No internet, wrong credentials, or more slots enabled than the hardware can service (each TLS link costs roughly 40 KB of heap). |
| Version has no `-exp` suffix | You're running different firmware than you think — likely the flash didn't take. Reflash. |

To capture logs for a bug report:

```bash
~/.platformio-venv/bin/pio device monitor -p /dev/cu.usbmodem1101 -b 115200 | tee observer.log
```

---

## Returning to official firmware

This firmware is not special — flash official MeshCore firmware over it at any
time using the [MeshCore web flasher](https://flasher.meshcore.io) or by
building an official environment from the upstream repository.

Your settings (WiFi credentials, MQTT slots, node identity) live in a separate
flash region and **survive reflashing**. They persist across firmware changes
unless you explicitly erase them:

```bash
~/.platformio-venv/bin/pio run -e heltec_v4_r8_room_server_observer_mqtt -t erase
```

That wipes **everything**, including your node's identity and keys. Your node will
appear as a brand new device to the mesh afterwards. There is no undo.

To remove the build tooling entirely:

```bash
rm -rf ~/.platformio-venv ~/.platformio
```

---

## Getting help

Include all of the following, or the answer will just be a request for it:

- Output of `ver` (confirming the `-exp` suffix)
- The exact environment name you built
- Your OS and version
- The complete error output, not a summary
- Console log from boot onward

Bear in mind these are **unofficial, AI-generated, experimental builds**. Please
don't take problems with them to the MeshCore project or to Heltec — neither is
responsible for this firmware. Report issues at
<https://github.com/tgodfrey/MeshCore/issues>.

---

*Provided as-is, without warranty or guarantee of any kind. All use at your own
risk.*
