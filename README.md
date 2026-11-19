# OpenDrop custom firmware (nanoSYD)

Custom firmware for the **OpenDrop V4.2** digital-microfluidics platform.

This firmware was originally part of the [OpenDrop](https://github.com/GaudiLabs/OpenDrop) project by
**Urs Gaudenz, GaudiLabs**. This fork is maintained by **Mikolaj Ostrowski** at
**nanoSYD, Syddansk Universitet (University of Southern Denmark)**, not by GaudiLabs.
Please report issues with this version here, not to GaudiLabs.

- Maintainer: Mikolaj Ostrowski · mikolaj.p.ostrowski@gmail.com · [github.com/ostryJR](https://github.com/ostryJR)
- Current version: **1.0.0** (based on GaudiLabs OpenDrop firmware V42.11)

## What's different from the stock firmware

**PC control over USB serial**
- Set high voltage (50–240 V), AC frequency (100–1000 Hz) and AC/DC mode from the PC app, not only from the device menu.
- Turn electrode feedback on/off from the app.
- "Save to device" command stores the current settings in flash.
- Show a short text message (up to 20 characters) on the device's OLED.
- The COM port keeps working while the on-device menu is open.
- Faster serial handling (all pending bytes are processed at once).
- Full byte-level description: [`customFirmware/SERIAL_PROTOCOL.md`](customFirmware/SERIAL_PROTOCOL.md).

**Device**
- Menu: the SET row is gone; leaving the menu after a change asks **SAVE SETTINGS? YES / NO**.
- New boot screens: OpenDrop splash marked "MODIFIED FW" with the version, then a nanoSYD credit screen.
- New, softer startup and menu sounds; fixed sound playback (stock played a WAV header as audio and one sound at the wrong sample rate).
- Removed an unused SD-card library include that caused compiler warnings.

## Repository layout

```
CLAUDE.md                     notes for AI-assisted development (project facts)
OriginalSoftware/             unmodified GaudiLabs firmware, for reference
customFirmware/
  OpenDropVCustom/            the Arduino sketch (open OpenDropVCustom.ino)
  Libraries/
    OpenDrop/                 device library (display, HV, menu, settings)   - modified
    OpenDropAudio/            sound playback + sound data                     - modified
    Adafruit-GFX-Library/     third-party, unmodified
    Adafruit_SSD1306/         third-party, unmodified
    Adafruit_BusIO/           third-party, unmodified
  SERIAL_PROTOCOL.md          PC app <-> device protocol
  tools/wav2sound.py          converts .wav files to device sounds
```

## Building and flashing

1. Install the [Arduino IDE](https://www.arduino.cc/en/software) and, in the Boards Manager, the **Arduino SAMD Boards** core (OpenDrop V4 uses a SAMD21 microcontroller). Select the board as described in the GaudiLabs OpenDrop instructions.
2. Copy the folders in `customFirmware/Libraries/` into your Arduino libraries folder
   (macOS/Windows: `Documents/Arduino/libraries/`).
   **The IDE compiles the copies in that folder**, so copy them again after every change in this repo.
   If you have the stock OpenDrop libraries installed, replace them.
3. Open `customFirmware/OpenDropVCustom/OpenDropVCustom.ino`, connect the OpenDrop over USB, select its port and click **Upload**.
4. On boot the OLED shows `nS v1.0.0`, and the serial port prints
   `Welcome to OpenDrop (nanoSYD fw 1.0.0+gaudi.42.11)` at 115200 baud.

## Using it from a PC app

The device speaks a binary protocol at **115200 baud**: the app sends 32-byte frames
(16 electrode-column bytes + 16 control bytes) and gets a 24-byte reply to each one.
The new control bytes use 0 to mean "no change", so an app that leaves them at 0 behaves as with the stock firmware
(not yet verified with the stock GaudiLabs app).
See [`customFirmware/SERIAL_PROTOCOL.md`](customFirmware/SERIAL_PROTOCOL.md) for the byte map, rules and a Python example.

## Changing the sounds

Sounds are stored in flash as 16 kHz, 8-bit mono data (`Sound1.h` = menu, `Sound2.h` = startup).

```sh
# convert your own .wav (any rate/bit depth; trimmed, faded, normalised)
python3 customFirmware/tools/wav2sound.py my.wav sound2 customFirmware/Libraries/OpenDropAudio/src/Sound2.h --gain 0.5
# rebuild the default synthesised sounds
python3 customFirmware/tools/wav2sound.py --generate
```
MP3s can be converted first on macOS with `afconvert -f WAVE -d LEI16 in.mp3 out.wav`.
Keep the startup sound under about 1 s and the menu sound under 0.15 s (each second uses 16 KB of flash).

## Versioning

This fork uses [Semantic Versioning](https://semver.org): `MAJOR.MINOR.PATCH`.
- **MAJOR**: a serial-protocol change that breaks older PC apps
- **MINOR**: a new feature that stays compatible
- **PATCH**: a bug fix

The version is set in one place, `FW_VERSION` at the top of `OpenDropVCustom.ino`. Releases are tagged `vX.Y.Z` in git.

## Credits and license

- Original OpenDrop hardware and firmware: Urs Gaudenz, [GaudiLabs](https://www.gaudi.ch/OpenDrop/). The OpenDrop project is published under the GNU GPL v3; this fork follows the upstream licensing.
- Sound playback is based on the Arduino AudioZero library (Arduino LLC), flash storage on Arduino's FlashStorage (Cristian Maglie).
- Display libraries: Adafruit GFX, Adafruit SSD1306, Adafruit BusIO — each under its own license (see their folders).
- Custom version: Mikolaj Ostrowski, nanoSYD, Syddansk Universitet.
