# ESP32 VCR Remote

Firmware for an **ESP32 DOIT DevKit v1** that emulates the infrared remote for a
**Panasonic PV-V4525S** VCR (Omnivision 4-head Hi-Fi VHS, remote model
EUR7723KA0). Commands are issued over the USB serial port.

IR codes are the IRDB `Panasonic/VCR/144,0` codeset, verified against the
Remote Central Pronto codes for the sibling PV-V4520. Transmission uses raw
Pronto (`IRsend::sendPronto`) rather than `encodePanasonic`, because the
former was empirically found to work on this specific VCR.

## Hardware

- **Board**: ESP32 DOIT DevKit v1 (or any ESP32 — update `upload_port` in
  `platformio.ini`).
- **IR LED driver**: GPIO 4 → 2N3904 base (~470 Ω) → NPN low-side switch.
  LED anode to Vcc through a ~100 Ω current-limit resistor, cathode to
  collector, emitter to GND. Reference circuit:
  https://github.com/crankyoldgit/IRremoteESP8266/wiki#ir-sending
- Aim within ~6 ft of the VCR's IR window. A bare 2N3904-driven LED is weak;
  if a button doesn't register, move closer before suspecting the code.

## Build and upload

```
pio run                 # compile
pio run -t upload       # compile + flash
pio device monitor      # open serial (115200 baud)
```

The `upload_port` and `monitor_port` are pinned to `/dev/cu.usbserial-0001`
in `platformio.ini`. If your board enumerates elsewhere, edit those lines or
override on the command line:

```
pio run -t upload --upload-port /dev/cu.usbserial-XXXX
pio device monitor -p /dev/cu.usbserial-XXXX
```

## Sending commands via CLI

Open the monitor, type a command, press **Enter**. Commands are
case-insensitive.

```
pio device monitor
```

On connect you'll see:

```
Panasonic PV-V4525S VCR remote. Type a command + Enter.
commands: POWER, PLAY, STOP, REW, FF, EJECT, PAUSE
```

Then, at the prompt (there is no prompt character — just type):

```
power        # toggle VCR power
play         # start playback
stop         # stop tape
rew          # rewind
ff           # fast-forward
eject        # eject tape
pause        # pause playback
?            # (or "help") reprint the command list
```

Each accepted command prints `send <NAME>` and transmits the IR frame once.
Unknown input prints `unknown: <input>` followed by the help list.

### Exiting the monitor

`Ctrl+C` exits `pio device monitor`.

### Piping commands (non-interactive)

`pio device monitor` is interactive-only. To script commands, use any serial
CLI that speaks a TTY at 115200 8N1, for example:

```
# send POWER once and exit
printf 'power\n' > /dev/cu.usbserial-0001
```

Or with `picocom` / `screen`:

```
picocom -b 115200 /dev/cu.usbserial-0001
screen /dev/cu.usbserial-0001 115200
```

Note: PlatformIO's monitor holds the port exclusively while open. Close it
before scripting against the raw device node.

## Adding a new command

1. Look up the function code in the IRDB CSV:
   https://raw.githubusercontent.com/probonopd/irdb/master/codes/Panasonic/VCR/144,0.csv
2. Generate a Pronto frame. Given `device=144`, `subdevice=0`, and a function
   byte `fn`, the checksum is `device ^ subdevice ^ fn`. Each byte is
   transmitted LSB-first. The full frame is 48 bits wrapped in the standard
   Kaseikyo/Panasonic Pronto header/leader/trailer. See
   `kIr_POWER` in `src/main.cpp` for the layout, or copy the generator used
   at build time.
3. Add the new `kIr_<NAME>[]` array and append `CMD(<NAME>)` to the
   `kCommands[]` table. The dispatcher picks it up automatically.
