---
slug: rooftop-sunrise-sunset-cam
title: Rooftop Sunrise and Sunset Camera — Automated Solar Time-Lapse
description: Two ESP32-CAMs on the roof, one facing east, one west. Both wake at sunrise and sunset, take a photo, and post it to Instagram automatically. No mains power. No manual effort after setup.
date: 2026-05-05
category: ESP32
status: Planning
tags: [ESP32-CAM, Solar, Deep Sleep, WiFi, Python, Instagram]
youtube:
parts:
  - name: AI Thinker ESP32-CAM
    qty: 2
    price: "already owned"
    notes: One east-facing, one west-facing
  - name: Solar panel (5V)
    qty: 2
    price: "$8 each"
  - name: LiPo battery
    qty: 2
    price: "$10 each"
  - name: LiPo charge controller
    qty: 2
    price: "$3 each"
  - name: Weatherproof enclosure (3D printed)
    qty: 2
    price: "filament"
    notes: Custom designed, camera lens port, solar panel mount
  - name: Roof mounting bracket (3D printed)
    qty: 2
    price: "filament"
---

## The idea

Two ESP32-CAMs sitting on the roof, one pointed east, one pointed west. Both running on solar power. Both connected to home WiFi.

Every morning the east camera wakes at sunrise, takes a photo, and sends it to a server on the home network. Every evening the west camera does the same at sunset. A Python script on the server posts both photos to Instagram automatically.

The feed fills itself. No manual effort after initial setup.

## Why this works

The roof has everything this needs:

- Full sun exposure all day — solar is ideal
- Home WiFi reach — no separate network required
- Clear east and west horizons — unobstructed sunrise and sunset views
- Out of the way — nothing to knock over or interfere with

The ESP32-CAM spends almost all its time in deep sleep. It wakes briefly, takes one shot, sends it over WiFi, then goes back to sleep. A small solar panel handles the power budget easily.

## Wakeup timing

Sunrise and sunset times are calculated on-chip from latitude and longitude — not hardcoded seasonal tables. Hardcoded times mean a firmware update every time you move the camera, which defeats the "drop it anywhere" appeal. The maths is spherical trig, a few hundred lines, no library required. Smaller libs exist if needed: SunRise, sunset.h.

## The automation chain

1. ESP32-CAM wakes at calculated sunrise or sunset time
2. Camera takes a JPEG and stores it to SD card as backup
3. ESP32-CAM connects to home WiFi and HTTP POSTs the image to a home server
4. Python script on the server receives the image
5. Script posts to Instagram with an auto-generated caption — date, time, sunrise or sunset

## Instagram posting

The official Instagram Graph API is the right call for a feed you want to last. It requires a Creator or Business account linked to a Facebook Page and a registered app, but it is stable and Meta-sanctioned.

The unofficial instagrapi library is simpler to set up but Meta actively detects automation patterns and suspends accounts that trigger them. Fine for a personal experiment, not for a feed you want running unattended for years.

Caption generated automatically from timestamp: `Sunrise — 05 May 2026, 06:47`

## ESP32-CAM GPIO note

The AI Thinker ESP32-CAM shares several GPIOs between the SD card and other peripherals. Use SD_MMC in 1-bit mode — it frees GPIO 4, 12, and 13, which you will need for other things. Worth knowing going in rather than discovering it mid-debug.

## Power budget

The camera plus WiFi upload is a meaningfully higher power draw than the weather station. A postage-stamp panel is not enough here — plan for a larger panel and battery. Winter is the unforgiving season: sunrise and sunset both happen when the panel is at its least productive angle, and the days are short.

## Hardware

Both cameras are already owned — AI Thinker ESP32-CAMs sitting unused. Enclosures will be custom 3D printed with a lens port, drainage, and a solar panel mount.

## Status

Planning. Cameras on hand. Enclosure not designed. Server-side script not written.
