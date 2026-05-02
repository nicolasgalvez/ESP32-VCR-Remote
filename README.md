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
pio run -t upload -b <BOARD>       # compile + flash
pio device monitor  -b 115200    # open serial (115200 baud)
```

The `upload_port` and `monitor_port` are pinned to `/dev/cu.usbserial-0001`
in `platformio.ini`. If your board enumerates elsewhere, edit those lines or
override on the command line:

```
pio run -t upload --upload-port /dev/cu.usbserial-XXXX
pio device monitor -p /dev/cu.usbserial-XXXX -b 115200
```

## Sending commands via CLI

Open a serial connection at **115200 8N1**, type a command, press **Enter**.
Commands are case-insensitive.

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

### Interactive: any of these work

The device path differs between platforms — on macOS it's
`/dev/cu.usbserial-0001`, on Linux usually `/dev/ttyUSB0`. Substitute as
needed.

```
# PlatformIO (uses upload_port from platformio.ini)
pio device monitor

# picocom — exit with Ctrl+A then Ctrl+X
picocom -b 115200 /dev/cu.usbserial-0001

# GNU screen — exit with Ctrl+A then k, then y
screen /dev/cu.usbserial-0001 115200

# minicom — exit with Ctrl+A then x
minicom -b 115200 -D /dev/cu.usbserial-0001

# cu (BSD/macOS) — exit by typing ~. on its own line
cu -l /dev/cu.usbserial-0001 -s 115200
```

### Non-interactive: scripting one-shot commands

Configure the port once with `stty`, then `printf` to the device node. The
firmware reads until `\n`, so newline-terminate every command.

macOS:

```
stty -f /dev/cu.usbserial-0001 115200 cs8 -cstopb -parenb -ixon raw
printf 'power\n' > /dev/cu.usbserial-0001
sleep 1
printf 'play\n'  > /dev/cu.usbserial-0001
```

Linux:

```
stty -F /dev/ttyUSB0 115200 cs8 -cstopb -parenb -ixon raw
printf 'power\n' > /dev/ttyUSB0
```

To capture the firmware's response while sending, hold the port open for
reading in one shell and write from another:

```
# shell A: read
cat /dev/cu.usbserial-0001

# shell B: write
printf 'power\n' > /dev/cu.usbserial-0001
```

Note: serial ports are exclusive on macOS/Linux. Close `pio device monitor`,
`screen`, `picocom`, etc. before scripting against the raw device node, or
the writes will silently go nowhere.

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
