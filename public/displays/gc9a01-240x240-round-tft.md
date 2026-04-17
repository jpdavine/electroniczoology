# 1.28" Round TFT Display (GC9A01) — ESP32 Dev Board

**Display:** 1.28" round TFT, 240×240, labelled "GC9A01 VER1.0"  
**Board:** ESP32 Dev Board (38-pin)  
**Library:** Arduino_GFX (GFX Library for Arduino)  
**Status:** CONFIRMED WORKING

---

## What You Need

### Parts
- ESP32 Dev Board (38-pin)
- 1.28" round TFT display (GC9A01, 240×240)
- Jumper wires
- USB cable for flashing

### Software
- Arduino IDE
- ESP32 board support installed
- Arduino_GFX library (v1.6.1 or later)
  - Install via Arduino IDE: Sketch → Include Library → Manage Libraries → search "GFX Library for Arduino"

---

## Wiring

| Display Pin | ESP32 GPIO | Notes |
|-------------|------------|-------|
| SCL | 18 | SPI clock |
| SDA | 23 | SPI data (MOSI) |
| DC | 27 | Data/Command select |
| CS | 5 | Chip select |
| RST | 4 | Reset |
| VCC | 3.3V | 3.3V only — do not use 5V |
| GND | GND | |
| BL | 3.3V | Backlight — wire direct for always-on |

> **Why GPIO27 for DC?** GPIO2 is a strapping pin on the ESP32 — holding it HIGH or LOW at boot can prevent uploads. GPIO27 is safe.

---

## A Note on the Chip Label

This display is sold as "GC9A01" and that's what it actually is — a genuine 240×240 round display.

> If you have a **smaller 160×160** round display also labelled "GC9A01", be careful — that one is likely a **GC9D01** with the wrong label. They need a different driver. See the GC9D01 tutorial.

---

## The Sketch

```cpp
/*
 * 1.28" GC9A01 Round TFT — ESP32 Dev Board
 * ==========================================
 * SCL  → GPIO18   SDA  → GPIO23
 * DC   → GPIO27   CS   → GPIO5
 * RST  → GPIO4    VCC  → 3.3V
 * GND  → GND      BL   → 3.3V
 */

#include <Arduino_GFX_Library.h>

Arduino_DataBus *bus = new Arduino_HWSPI(27, 5); // DC, CS
Arduino_GFX *gfx = new Arduino_GC9A01(bus, 4, 0, true); // RST, rotation, IPS=true

void setup() {
  gfx->begin();
  gfx->fillScreen(0x0000); // black

  gfx->setTextColor(0xFFFF); // white
  gfx->setTextSize(3);
  gfx->setCursor(60, 100);
  gfx->println("HELLO");

  gfx->setTextColor(0xF800); // red
  gfx->setTextSize(2);
  gfx->setCursor(50, 140);
  gfx->println("GC9A01 OK");
}

void loop() {}
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

> **Gotcha:** Arduino_GFX does not define colour name constants like `BLACK` or `WHITE` by default. Always use the hex values above — using names will cause a compile error.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Colours look inverted | Change `true` to `false` in `Arduino_GC9A01(bus, 4, 0, true)` |
| Blank screen | Check wiring, especially DC and CS. Confirm 3.3V not 5V |
| Upload fails | Make sure DC is not wired to GPIO2 — use GPIO27 |
| Text or shapes don't appear | Check `gfx->begin()` is called before drawing |

---

## What's Next

- Drawing shapes (circles, lines, filled rectangles)
- Displaying bitmaps / images
- Animating content with `millis()` timers
- Using this display with the tank rover project
