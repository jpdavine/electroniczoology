# PAM8403 Audio Amplifier — ESP32 Dev Board

**Amp:** PAM8403 (3W stereo class-D)  
**Board:** ESP32 Dev Board (38-pin)  
**Speaker:** Any 4–8Ω speaker (laptop speaker works great)  
**Status:** CONFIRMED WORKING

> Commands in this guide are shown in code blocks. Run them in a terminal — on Linux that's whatever terminal emulator you use (xterm, Alacritty, GNOME Terminal, etc.).

---

## What You Need

### Parts
- ESP32 Dev Board (38-pin)
- PAM8403 amplifier module
- 4–8Ω speaker
- 10µF electrolytic capacitor (coupling cap — signal path)
- 100µF electrolytic capacitor (decoupling cap — power supply)
- Jumper wires
- USB cable for flashing

### Software
- Arduino IDE with ESP32 board support
- `espeak-ng` — for generating voice clips
- `ffmpeg` — for converting audio files
- Python 3 — for converting WAV to C header

Install on Arch Linux:
```
sudo pacman -S espeak-ng ffmpeg python
```

---

## How It Works

The ESP32 has two built-in DAC pins (GPIO25 and GPIO26) that output a true analogue voltage (0–3.3V). We use one of these to output audio samples, which the PAM8403 amplifies to drive the speaker.

The audio data is stored as a C byte array compiled directly into the sketch — no SD card or filesystem needed for short clips.

---

## Wiring

| From | To |
|---|---|
| ESP32 GPIO25 → 10µF cap (+) | PAM8403 `L` |
| 10µF cap (−) | PAM8403 `G` |
| ESP32 5V | PAM8403 VCC |
| ESP32 GND | PAM8403 GND |
| PAM8403 L OUT+ | Speaker + |
| PAM8403 L OUT− | Speaker − |

Also place a 100µF cap across PAM8403 VCC and GND (+ leg to VCC).

**PAM8403 input pins:** The input side has three pins — `L` (left), `G` (ground reference), and `R` (right). For mono use, connect your signal to `L` and ground to `G`. Leave `R` unconnected.

> **Why the 10µF coupling cap?** The ESP32 DAC output is centred at ~1.65V (not 0V). Without the cap, that DC offset passes straight into the amplifier and causes a loud thump on power-on and distorts the audio. The cap blocks DC and passes only the AC audio signal.

> **Why the 100µF decoupling cap?** The PAM8403 draws sudden bursts of current when playing audio. The cap sits on the power rail and smooths those spikes, reducing hiss and noise in the output.

> **No volume pot on your module?** Control volume in code — see the sketch below.

---

## Preparing an Audio Clip

### Option 1 — Generate a voice clip with espeak-ng

```
espeak-ng -w /tmp/myclip.wav "your text here"
```

### Option 2 — Extract from an existing audio/video file

Use ffmpeg to cut a section from any WAV, MP3, or video file:

```
ffmpeg -i /path/to/source.wav -ss 00:00:05 -to 00:00:08 /tmp/myclip.wav
```

**Flags:**
- `-i` — input file (replace with your actual path)
- `-ss 00:00:05` — start at 5 seconds
- `-to 00:00:08` — end at 8 seconds (3-second clip)

**Finding timestamps:** Open the file in VLC, note where your clip starts and ends, and use those values for `-ss` and `-to`.

**Examples:**

```bash
# File in home folder
ffmpeg -i ~/mysound.wav -ss 00:00:05 -to 00:00:08 /tmp/myclip.wav

# File in Downloads
ffmpeg -i ~/Downloads/mysound.mp3 -ss 00:00:05 -to 00:00:08 /tmp/myclip.wav

# Video file — ffmpeg extracts audio automatically
ffmpeg -i ~/Videos/myvideo.mp4 -ss 00:00:05 -to 00:00:08 /tmp/myclip.wav

# Filename with spaces — wrap in quotes
ffmpeg -i "/home/j/My Audio File.wav" -ss 00:00:05 -to 00:00:08 /tmp/myclip.wav
```

### Convert to ESP32-friendly format

The ESP32 DAC works best at 8kHz, 8-bit, mono. Run this as a single command:

```
ffmpeg -y -i /tmp/myclip.wav -ar 8000 -ac 1 -acodec pcm_u8 /tmp/myclip_8bit.wav
```

**Flags:**
- `-y` — overwrite output without prompting
- `-ar 8000` — 8000 samples per second
- `-ac 1` — mono
- `-acodec pcm_u8` — 8-bit unsigned PCM (0–255, 128 = silence) — matches what `dacWrite()` expects

> **Why 8kHz?** It's what phone calls use — speech is fully intelligible. A 1-second clip at 8kHz = 8KB; at 44.1kHz = 44KB. The ESP32 has ~4MB flash, so you have headroom, but 8kHz is a sensible default.

---

## Convert WAV to C Header

Save this as `wav_to_header.py`:

```python
import sys

input_file = sys.argv[1]   # e.g. /tmp/myclip_8bit.wav
output_file = sys.argv[2]  # e.g. myclip.h
var_name = sys.argv[3]     # e.g. myclip

with open(input_file, 'rb') as f:
    data = f.read()

pcm = data[44:]  # skip 44-byte WAV header

lines = [
    '#pragma once',
    '#include <pgmspace.h>',
    f'const uint32_t {var_name}_len = {len(pcm)};',
    f'const uint8_t {var_name}_data[] PROGMEM = {{'
]

row = []
for b in pcm:
    row.append(f'0x{b:02X}')
    if len(row) == 16:
        lines.append('  ' + ', '.join(row) + ',')
        row = []
if row:
    lines.append('  ' + ', '.join(row))
lines.append('};')

with open(output_file, 'w') as f:
    f.write('\n'.join(lines) + '\n')

print(f"Done: {len(pcm)} bytes → {output_file}")
```

Then run:
```
python3 wav_to_header.py /tmp/myclip_8bit.wav myclip.h myclip
```

Place `myclip.h` in your sketch folder — the same directory as your `.ino` file.

---

## Sketch

```cpp
// PAM8403 audio playback
// GPIO25 → 10µF cap → PAM8403 L

#include "myclip.h"  // generated from wav_to_header.py

const int DAC_PIN     = 25;
const int SAMPLE_RATE = 8000;
const float VOLUME    = 0.8;  // 0.0 = silent, 1.0 = max

void playAudio() {
  uint32_t intervalUs = 1000000 / SAMPLE_RATE;
  for (uint32_t i = 0; i < myclip_len; i++) {
    uint8_t sample = pgm_read_byte(&myclip_data[i]);
    // Scale around midpoint to control volume
    int val = 128 + (int)((sample - 128) * VOLUME);
    val = constrain(val, 0, 255);
    dacWrite(DAC_PIN, val);
    delayMicroseconds(intervalUs);
  }
  dacWrite(DAC_PIN, 128); // return to midpoint (silence)
}

void setup() {
  playAudio();
}

void loop() {
  // Nothing here — clip plays once on boot.
  // Reset the ESP32 to play again.
}
```

> **PROGMEM** stores the audio data in flash instead of RAM. Without it, a large audio array would overflow the ESP32's 320KB RAM. `pgm_read_byte()` reads it back one byte at a time during playback.

> **Volume control:** Samples are 8-bit unsigned (0–255) centred at 128. To reduce volume, scale the offset from that midpoint — e.g. at VOLUME=0.5, a sample of 200 becomes 128 + (200−128)×0.5 = 164. Start at 0.3 and work up.

---

## Troubleshooting

**No sound at all**
- Check 5V and GND are connected to the PAM8403
- Check speaker is on L OUT+ and L OUT−
- If your module has an SD (shutdown) pin, tie it HIGH to 3.3V

**Very quiet**
- Increase `VOLUME` in the sketch
- Check the 10µF cap is the right way around (+ leg toward ESP32)

**Hiss or noise**
- Add or increase the 100µF decoupling cap on VCC/GND
- Make sure GND is shared between ESP32 and PAM8403

**Wrong playback speed**
- `SAMPLE_RATE` in the sketch must match the ffmpeg `-ar` value (both should be `8000`)

**Clip too large / flash usage too high**
- Trim the clip shorter with ffmpeg
- Drop to 4kHz: `-ar 4000` — speech stays intelligible at half the size
