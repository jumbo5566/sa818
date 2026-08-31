# USBaudio — CM108 Sound Card PTT Controller + SA818/SR-FRS Programming Tool

A Windows command-line tool for amateur radio applications, built around a
**CM108-family USB sound card** (`USB\VID_0D8C&PID_0012`) and a **CH340
USB-serial adapter** (`USB\VID_1A86&PID_7523`). It provides two main features:

1. **PTT control** — monitors the playback volume of the USB sound card and
   keys the radio transmitter (PTT) automatically
2. **Radio programming** — programs frequency, tone, squelch and volume on
   SA818 / SR-FRS modules over the serial port

Implemented with pure Win32 APIs (WASAPI / HID / SetupAPI / serial port).
**Statically linked, single-file distribution, no DLLs required.**

---

## Modes

| Command | Mode | Description |
|---|---|---|
| `recorder.exe` | **Auto mode (default)** | Picks a mode by detected hardware, see below |
| `recorder.exe -f` | Frequency setup | Find the CH340 serial adapter, detect SA818 / SR-FRS, program the radio |
| `recorder.exe -t` | Test mode | Record 5 s from the USB mic → normalize → play back 5 s, verifies the PTT chain |
| `recorder.exe -r` | Recorder mode | Continuous recording from the USB mic, press `q` to stop, saves a WAV and draws the waveform |
| `recorder.exe -h` | Help | Show all options |

### Auto mode decision chain

```
recorder.exe (no arguments)
   ├─ CM108 USB sound card present? ──yes──> PTT mode
   ├─ CH340 serial adapter present?  ──yes──> Frequency setup mode
   └─ neither found                  ──>     show help and exit
```

---

## How PTT mode works

```
USB sound card plays audio (e.g. AFSK/FT8 output from digimode software)
        │
        ▼  WASAPI loopback capture of the render endpoint
   Real-time volume monitoring (every 100 ms, with a level bar)
        │
   volume > 5%  ──> PTT ON  (GPIO3 high, minimum 3 s hold)
   volume ≤ 5%  ──> PTT OFF (after the 3 s hang time)
        │
        ▼
   CM108 HID report → GPIO3 → radio PTT
```

- **GPIO3 (bit 2)**: PTT control. Volume above 5 % keys the transmitter with a
  3-second hang time for continuous transmit.
- **IO4 (bit 3)**: driven high for 5 seconds at startup, then low
  (one-shot pulse) — this is the **blue COS LED** on the board.
- Live status line: `[  3.2s] TX vol [########------]  35%  PTT:ON  'q' quit`
- Press `q` to quit; PTT and IO4 are released automatically on exit.

> ⚠️ **Note (Windows)**: when using the "incoming radio signal triggers on
> volume decrease" feature, please remove the diode on the board.

## Frequency setup mode

The CH340 serial adapter (e.g. COM23) is found automatically and the module
type is **auto-detected**:

| | SA818 | SR-FRS |
|---|---|---|
| Version command | `AT+VERSION` | `AT+DMOVERQ` |
| Set-group command | `AT+DMOSETGROUP=bw,tx,rx,tx_tone,sq,rx_tone` | `AT+DMOSETGROUP=bw,tx,rx,tone,sq,tone,1` |
| Tone encoding | 4 digits (CTCSS `0008` / DCS `023N`) | 2 digits (CTCSS `08`) |

Interactive flow (enter `q` at any prompt to abort without writing):

```
RX frequency MHz [145.5000]:  144.64        ← auto-padded to 4 decimals → 144.6400
TX frequency MHz [same as RX]:               ← Enter = same as RX
Tone: 0=none 1=CTCSS 2=DCS [0]: 1
CTCSS Hz (e.g. 71.9): 88.5
Squelch 1-8 [8]:
Volume 1-8 [8]:
Bandwidth 0=narrow(12.5k) 1=wide(25k) [0]: 1
[SA818] SETGROUP reply: +DMOSETGROUP:0       ← programmed successfully
```

- **No frequency range checking** is done by the tool (the module itself
  rejects out-of-range frequencies, e.g. a UHF frequency on a VHF module
  returns `+DMOSETGROUP:1`).
- Built-in table of the 38 standard CTCSS tones; entering `88.5` is converted
  to the module's tone code automatically.
- DCS input is a 3-digit code plus N/I polarity (note: the SR-FRS firmware
  `110V-V223` tested here does not accept DCS — use CTCSS).

## Test mode

One command verifies the whole record → playback → PTT chain, about 15 s,
fully automatic:

1. Record 5 seconds from the USB microphone
2. **Normalize to 80 % peak** (even a very quiet mic still plays back loud
   enough to trigger PTT)
3. Save `test.wav` and play it back on the USB speaker for 5 seconds
4. PTT follows the playback volume automatically
5. If the recording is silent, detailed troubleshooting hints are printed
   (mic level / privacy settings / wiring)

---

## Hardware hookup

- **CM108 sound card** (C-Media, VID 0D8C / PID 0012):
  - Speaker output → radio audio input (transmit audio)
  - Mic input ← radio audio output (optional, for recording received audio)
  - GPIO3 (CM108AH pin 13) → radio PTT (some circuits need GPIO low for TX)
  - IO4 → on-board blue COS LED
- **CH340 serial adapter** (VID 1A86 / PID 7523) → SA818 / SR-FRS module
  (9600 baud, 8N1)

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `[PTT] No HID device found` | The CM108 HID interface is held by another program (e.g. SDR software), or the VID/PID differs |
| `+DMOSETGROUP:1` | Frequency outside the module's band (e.g. UHF on a VHF module) or wrong tone format |
| Recording is pure silence | Check Windows mic level / privacy settings and the CM108 mic input wiring |
| No sound on playback | Make sure the USB speaker endpoint is not disabled |

---

## License

Provided as-is for amateur radio use. The CM108 PTT implementation is a
Windows port of the open-source cm108-ptt (Hamlib-derived) code; the SR-FRS
protocol follows the srfrs.py reference by Fred (W6BSD) / BG7IYN.
