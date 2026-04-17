---
slug: pam8403-esp32-audio
title: PAM8403 Audio Amplifier with ESP32
description: Playing audio clips from an ESP32 dev board through a PAM8403 amplifier and a speaker. Covers wiring, audio prep, WAV-to-header conversion, and the full Arduino sketch.
date: 2026-04-16
category: Audio
tags: [ESP32, PAM8403, Audio, Arduino]
youtube:
parts:
  - name: ESP32 Dev Board (38-pin)
    qty: 1
    price: "$8"
    link: ""
  - name: PAM8403 amplifier module
    qty: 1
    price: "$3"
    link: ""
  - name: 4-8Ω speaker
    qty: 1
    price: "$5"
    link: ""
  - name: 10µF electrolytic capacitor
    qty: 1
    price: "$1"
    notes: Coupling cap - signal path
    link: ""
  - name: 100µF electrolytic capacitor
    qty: 1
    price: "$1"
    notes: Decoupling cap - power rail
    link: ""
  - name: Jumper wires
    qty: 1
    price: "$3"
    link: ""
---

## How it works

The ESP32 has two built-in DAC pins (GPIO25 and GPIO26) that output a true analogue voltage (0-3.3V). We use one of these to output audio samples, which the PAM8403 amplifies to drive the speaker.

The audio data is stored as a C byte array compiled directly into the sketch - no SD card or filesystem needed for short clips.

## Wiring

| From | To |
|------|-----|
| ESP32 GPIO25 → 10µF cap (+) | PAM8403 L |
| 10µF cap (-) | PAM8403 G |
| ESP32 5V | PAM8403 VCC |
| ESP32 GND | PAM8403 GND |
| PAM8403 L OUT+ | Speaker + |
| PAM8403 L OUT- | Speaker - |

Also place a 100µF cap across PAM8403 VCC and GND (+ leg to VCC).

## The sketch

```cpp
#include "myclip.h"

const int DAC_PIN     = 25;
const int SAMPLE_RATE = 8000;
const float VOLUME    = 0.8;

void playAudio() {
  uint32_t intervalUs = 1000000 / SAMPLE_RATE;
  for (uint32_t i = 0; i < myclip_len; i++) {
    uint8_t sample = pgm_read_byte(&myclip_data[i]);
    int val = 128 + (int)((sample - 128) * VOLUME);
    val = constrain(val, 0, 255);
    dacWrite(DAC_PIN, val);
    delayMicroseconds(intervalUs);
  }
  dacWrite(DAC_PIN, 128);
}

void setup() {
  playAudio();
}

void loop() {}
```

See the full guide for audio prep, WAV-to-header conversion, and troubleshooting.
