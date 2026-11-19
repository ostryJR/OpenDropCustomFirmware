# OpenDrop custom firmware (nanoSYD, Syddansk Universitet)

Custom firmware for the **OpenDrop V4.2** digital-microfluidics board (GaudiLabs, SAMD21 / Arduino Zero-class, 128x64 SSD1306 OLED).
Maintained by Mikolaj Ostrowski for nanoSYD. Remote: `github.com/ostryJR/OpenDropCustomFirmware` (branch `main`).

## Layout
- `OriginalSoftware/` — untouched stock GaudiLabs firmware, for reference/diffing. Don't edit.
- `customFirmware/OpenDropVCustom/OpenDropVCustom.ino` — the sketch (setup, main loop, serial protocol in `handleSerial()`).
- `customFirmware/Libraries/OpenDrop/` — device library: `OpenDrop.cpp` (display, HV, menu `Menu()`, flash settings, boot screens), `hardware_def.h` (pins, flags).
- `customFirmware/Libraries/OpenDropAudio/src/` — sound playback; `Sound1.h` = menu, `Sound2.h` = startup, `Sound3.h` = drop-move tick.
- `customFirmware/Libraries/Adafruit_*` — vendored third-party libs, unmodified.
- `customFirmware/SERIAL_PROTOCOL.md` — **the byte-level PC-app protocol; keep it in sync with any protocol change.**
- `customFirmware/tools/wav2sound.py` — converts .wav → sound header (16 kHz, unsigned 8-bit mono); `--generate` rebuilds the default synthesised chime/blip.

## Build / deploy facts
- **The Arduino IDE builds from `~/Documents/Arduino/libraries/{OpenDrop,OpenDropAudio}`, not from the repo.** After editing anything in `customFirmware/Libraries/OpenDrop*`, copy the changed files there (`diff -rq` to check sync).
- No Arduino toolchain on this Mac → changes can't be compiled or flashed here; the user builds/flashes in the IDE. Say so when reporting.
- Sketch folder name must match the `.ino` name (Arduino rule).
- Displayed version = `version_str` (`OpenDrop.cpp`, "V42.11.") + code passed to `OpenDropDevice.begin("c10")` in the sketch.

## What's custom vs stock (all on `main`)
- **Serial control** (32-byte frames, 24-byte reply): HV voltage `[13]`, freq/10 `[14]`, DC/AC `[15]` (0 = keep), feedback `[8]`, save-to-flash `[9]=1`, OLED message `[1]` (length N + N ASCII bytes after the frame, 255 = clear). Details in `SERIAL_PROTOCOL.md`.
- `OpenDrop::save_settings()` — writes HV/sound/feedback to flash only if changed; used by serial save and the menu.
- `OpenDrop::show_message()` — banner drawn in `update_Display()` before the INSERT CARTRIDGE / UNDER VOLTAGE warnings.
- **Menu**: no SET row; leaving with changes shows **SAVE SETTINGS? YES/NO** (joystick left = YES, right = NO, button confirms).
- **Serial keeps working inside the menu**: `Menu()` calls `menu_idle_hook` (set to `handleSerial` in setup); while `in_menu`, settings bytes `[8],[9],[13..15]` are ignored and `update_Display()` is skipped.
- `handleSerial()` drains all pending bytes per call (stock read 1 byte per loop).
- **Boot**: OpenDrop splash + "MODIFIED FW" (2.5 s, `delay` in `OpenDrop::begin()`), then nanoSYD credit screen (stays during the sketch's `delay(3500)` in `setup()`).
- **Sounds**: synthesised startup chime and menu blip; `playMe()` uses `sizeof()` (stock had hardcoded lengths, a WAV header played as audio and an 11 kHz sound played at 16 kHz).
- Removed unused `#include <SD.h>` from `OpenDropAudio.h` (it caused SD-library compiler warnings).

## Conventions
- Keep changes minimal and in the existing (old Arduino) code style; this firmware is flashed to lab hardware.
- OLED text: built-in 6x8 font, ~21 chars per line at size 1; check that text fits (wrap happens at x+6 >= 128).
- Credit header (nanoSYD / Syddansk Universitet / Mikolaj Ostrowski) is in the sketch, `OpenDrop.cpp/.h`, `OpenDropAudio.cpp`; keep it.
- "commit and everything" (user's phrase) = commit, fast-forward the feature branch into `main`, push `main` to origin.
- Show OLED screens as half-block text art in the chat (not PNG files), rendered from the real glcdfont + splash bitmap.
