# 0.71" Round TFT Display (GC9D01) — ESP32 Dev Board

**Display:** 0.71" round TFT, 160×160, often mislabelled "GC9A01"  
**Board:** ESP32 Dev Board (38-pin)  
**Library:** Arduino_GFX (GFX Library for Arduino) v1.6.1+  
**Status:** CONFIRMED WORKING

---

## What You Need

### Parts
- ESP32 Dev Board (38-pin)
- 0.71" round TFT display (160×160)
- Jumper wires
- USB cable for flashing

### Software
- Arduino IDE
- ESP32 board support installed
- Arduino_GFX library **v1.6.1 or later**
  - Install via Arduino IDE: Sketch → Include Library → Manage Libraries → search "GFX Library for Arduino"

---

## The Chip Label Problem

This display is almost always sold as **"GC9A01"** — but the actual driver chip is **GC9D01**. They are different chips with different init sequences.

If you use the GC9A01 driver on a GC9D01 display you will get:
- Vertical coloured lines across the screen
- Wrong colours
- Or a blank screen

The fix: use `Arduino_GC9D01`, not `Arduino_GC9A01`. Arduino_GFX added native GC9D01 support in v1.6.1.

> **How to tell which you have:** If your round display is 0.71" / 160×160, it's almost certainly GC9D01. If it's 1.28" / 240×240, it's GC9A01.

---

## Wiring

| Display Pin | ESP32 GPIO | Notes |
|-------------|------------|-------|
| SCL | 18 | SPI clock |
| SDA | 23 | SPI data (MOSI) |
| DC | 27 | Data/Command select |
| CS | 5 | Chip select |
| RST | 4 | Reset |
| VCC | 3.3V | 3.3V only |
| GND | GND | |
| BL | 3.3V | Backlight — wire direct for always-on |
| MISO | not connected | Display is write-only |

> **Why GPIO27 for DC?** GPIO2 is a strapping pin — it can prevent uploads if held HIGH or LOW at boot. GPIO27 is safe.

---

## The Sketch

```cpp
/*
 * 0.71" GC9D01 Round TFT — ESP32 Dev Board
 * ==========================================
 * Sold as "GC9A01" but actual chip is GC9D01 — use Arduino_GC9D01 driver.
 *
 * SCL  → GPIO18   SDA  → GPIO23
 * DC   → GPIO27   CS   → GPIO5
 * RST  → GPIO4    VCC  → 3.3V
 * GND  → GND      BL   → 3.3V
 * MISO → not connected
 */

#include <Arduino_GFX_Library.h>

Arduino_DataBus *bus = new Arduino_HWSPI(27, 5); // DC, CS
Arduino_GFX *gfx = new Arduino_GC9D01(bus, 4, 0, false); // RST, rotation, IPS=false

void setup() {
  gfx->begin();
}

void loop() {
  gfx->fillScreen(0xF800); delay(500); // red
  gfx->fillScreen(0x07E0); delay(500); // green
  gfx->fillScreen(0x001F); delay(500); // blue
  gfx->fillScreen(0xFFFF); delay(500); // white
  gfx->fillScreen(0x0000); delay(500); // black
}
```

### Colour Reference (RGB565)
| Colour | Hex |
|--------|-----|
| Black | `0x0000` |
| White | `0xFFFF` |
| Red | `0xF800` |
| Green | `0x07E0` |
| Blue | `0x001F` |
| Yellow | `0xFFE0` |

> **Gotcha:** Do not use colour name constants like `BLACK` or `RED` — they are not defined by Arduino_GFX. Use hex values.

---

## How the Resolution Works

The GC9D01 controller has an internal 240×240 frame buffer, but the physical display is only 160×160. The driver automatically applies the 40-pixel offset so you don't need to set it manually — just draw as if the display is 160×160.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Vertical coloured lines | Wrong driver — switch from `Arduino_GC9A01` to `Arduino_GC9D01` |
| Colours inverted | Change `false` to `true` in the constructor (unlikely on this panel) |
| Blank screen | Check wiring, confirm 3.3V not 5V, check DC is not on GPIO2 |
| Upload fails repeatedly | DC pin may be on GPIO2 — move to GPIO27 |
| Library version error | Arduino_GFX must be v1.6.1 or later for GC9D01 support |

---

## What's Next

- Drawing shapes and text
- Displaying bitmaps
- Building the Diesel face display project with LittleFS + OTA
