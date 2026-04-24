# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

PlatformIO / Arduino-framework firmware for an **ESP32 DOIT DevKit v1** that drives an IR LED to act as a remote control (intended target: VCR, currently wired to send Panasonic TV power codes as a smoke test). IR transmission is handled by the `crankyoldgit/IRremoteESP8266` library — despite the name, it supports ESP32.

## Commands

All commands are run from the project root. PlatformIO provides the full toolchain; there is no separate build system.

- Build: `pio run`
- Upload to board: `pio run -t upload`
- Serial monitor: `pio device monitor`
- Clean build artifacts: `pio run -t clean`
- Unit tests (PlatformIO test runner): `pio test`
- Run a single test env/filter: `pio test -e esp32doit-devkit-v1 -f <test_name>`

The only build environment is `[env:esp32doit-devkit-v1]` in `platformio.ini`.

## Architecture notes

- `src/main.cpp` is the single translation unit. `setup()` initializes `IRsend` on `kIrLed` (GPIO 4) and `loop()` emits an IR command every 10 s.
- IR code transmission is gated on the `SEND_PANASONIC` compile-time flag exposed by `IRremoteESP8266`. Enabling or disabling additional protocols is done via that library's `IRremoteESP8266.h` defines — prefer toggling them with `build_flags` in `platformio.ini` over editing the library headers, so the change survives a lib reinstall.
- Hardware expectation: the IR LED is **not** driven directly from GPIO 4 — it must go through a transistor driver (see the comment header in `main.cpp` and the IRremoteESP8266 wiki). When changing `kIrLed`, avoid GPIO 0/1/3 (boot / UART0).
- `include/`, `lib/`, and `test/` currently contain only PlatformIO's default README stubs; no project headers, private libraries, or unit tests exist yet.

## Conventions

- Keep new IR protocol work behind the upstream `SEND_*` macros so disabling a protocol at build time still compiles.
- When adding private helpers, put shared declarations in `include/` and multi-file modules in `lib/<name>/` so PlatformIO's LDF picks them up automatically.
