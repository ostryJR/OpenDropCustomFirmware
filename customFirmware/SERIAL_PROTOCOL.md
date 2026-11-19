# OpenDrop custom firmware — serial protocol (for the PC app)

USB CDC serial, **115200 baud, 8N1**. Binary, no framing or sync bytes: the device simply counts bytes.
On boot the device prints its ID as text plus `"Welcome to OpenDrop (nanoSYD fw 1.0.0+gaudi.42.11)"` — the app can parse the firmware version (SemVer) from this line, then flush the input buffer. A different MAJOR version means an incompatible protocol.

## Request: app → device, exactly 32 bytes per frame (+ optional text)

| Byte(s) | Name | Meaning |
|---|---|---|
| 0–15 | Fluxels | One byte per **column x = 0..15**. Bit `y` (bit 0 = row 0 … bit 7 = row 7) = electrode on/off. |
| 16 | `ctrl[0]` | Magnets: bit 1 (`0x02`) = magnet 1, bit 0 (`0x01`) = magnet 2. |
| 17 | `ctrl[1]` | **Message on OLED**: `0` = no change, `1..254` = length N of text that follows the frame, `255` = clear message. |
| 18–23 | `ctrl[2..7]` | unused — send 0 |
| 24 | `ctrl[8]` | **Feedback**: `0` = off, any other value = on. Applied every frame. |
| 25 | `ctrl[9]` | **Save to flash**: `1` = store current voltage/freq/mode/feedback/sound permanently. Otherwise 0. |
| 26 | `ctrl[10]` | Temperature setpoint 1 (adapter) |
| 27 | `ctrl[11]` | Temperature setpoint 2 (adapter) |
| 28 | `ctrl[12]` | Temperature setpoint 3 (adapter) |
| 29 | `ctrl[13]` | **Voltage** in volts, 50–240 (clamped). `0` = keep current. |
| 30 | `ctrl[14]` | **Frequency ÷ 10**, 10–100 → 100–1000 Hz (clamped). `0` = keep current. |
| 31 | `ctrl[15]` | **Mode**: `1` = DC, `2` = AC. `0` = keep current. |
| 32… | text | Only if `ctrl[1]` = N in 1..254: exactly N ASCII bytes, sent **immediately** after byte 31 (within 1 s). |

Rules:
- Normal (continuous) frames: send `ctrl[1]=0`, `ctrl[9]=0`, `ctrl[13..15]=0` so nothing changes.
- `ctrl[0]` (magnets), `ctrl[8]` (feedback) and the fluxel bytes are **not** "0 = keep": they are applied on every frame. The app must always send its current state there (keep it in app state, also in message/save frames).
- HV values (`ctrl[13..15]`) take effect immediately when non-zero and different from current; they are **lost on reboot** unless saved.
- "Save to device" button: send **one** frame with `ctrl[13..15]` + `ctrl[8]` filled and `ctrl[9]=1`. The device applies first, then writes flash (only if something differs). Don't send `ctrl[9]=1` continuously.
- Message: max **20 characters shown** (longer text is read but truncated). Plain ASCII only (no accents). Message stays until replaced or cleared (`ctrl[1]=255`, no trailing bytes). "INSERT CARTRIDGE"/"UNDER VOLTAGE!" warnings are drawn over it.
- The N text bytes must always be sent in full, otherwise the next frame will be misaligned.

## While the device menu is open

The device keeps handling frames and sending replies while someone uses the on-device menu, but:
- Fluxels, magnets (`ctrl[0]`), temperatures (`ctrl[10..12]`) and the 24-byte reply work as normal.
- Feedback (`ctrl[8]`), Save (`ctrl[9]`) and HV (`ctrl[13..15]`) are **ignored** — the menu is editing those settings. Send them again after the menu closes (normal frames do this anyway for feedback).
- A message (`ctrl[1]`) is accepted and stored; it appears once the menu closes.
- Electrode feedback bits in the reply are not refreshed while the menu is open.

## Response: device → app, exactly 24 bytes after every frame

| Byte(s) | Meaning |
|---|---|
| 0–15 | Electrode state per column x = 0..15. **Bit order reversed vs. request: bit 7 = row 0 … bit 0 = row 7.** |
| 16 | unused (0) |
| 17, 18 | Temperature 1 low byte, high byte (from adapter) |
| 19, 20 | Temperature 2 low, high |
| 21, 22 | Temperature 3 low, high |
| 23 | Device ID |

## Python example

```python
import serial, time

s = serial.Serial('/dev/cu.usbmodemXXXX', 115200, timeout=2)
time.sleep(2); s.reset_input_buffer()

def frame(fluxels=None, magnets=0, feedback=0, save=0,
          volts=0, freq_hz=0, mode=0, message=None, clear_msg=False):
    flux = fluxels or [0]*16           # 16 ints, bit y = row y
    ctrl = [0]*16
    ctrl[0] = magnets
    ctrl[8] = 1 if feedback else 0
    ctrl[9] = 1 if save else 0
    ctrl[13] = volts                     # 0 = keep
    ctrl[14] = freq_hz // 10 if freq_hz else 0
    ctrl[15] = {'DC': 1, 'AC': 2}.get(mode, 0)
    tail = b''
    if clear_msg:
        ctrl[1] = 255
    elif message:
        tail = message.encode('ascii', 'replace')[:254]
        ctrl[1] = len(tail)
    s.write(bytes(flux + ctrl) + tail)
    return list(s.read(24))

frame(volts=200, freq_hz=500, mode='AC', feedback=1)            # apply only
frame(volts=200, freq_hz=500, mode='AC', feedback=1, save=1)    # "Save to device"
frame(message="PUMP DONE")                                      # show text
frame(clear_msg=True)                                           # clear text
```
