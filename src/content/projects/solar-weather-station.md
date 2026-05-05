---
slug: solar-weather-station
title: Solar-Powered ESP32 Weather Station
description: A fully wireless weather logger running on solar and battery power. Wakes once a day, takes readings, saves to NVS, serves data over WiFi. No mains power, goes anywhere.
date: 2026-05-05
category: ESP32
status: Planning
tags: [ESP32, Solar, Deep Sleep, NVS, WiFi, Sensors]
youtube:
parts:
  - name: ESP32 Dev Board
    qty: 1
    price: "$5"
    notes: Deep sleep mode for power efficiency
  - name: Solar panel (small, 5V)
    qty: 1
    price: "$8"
  - name: LiPo battery
    qty: 1
    price: "$10"
  - name: LiPo charge controller (TP4056 or CN3791)
    qty: 1
    price: "$3"
  - name: BME280 (temperature, humidity, pressure)
    qty: 1
    price: "$6"
  - name: Wind speed sensor (anemometer)
    qty: 1
    price: "$15"
  - name: Weatherproof enclosure (3D printed)
    qty: 1
    price: "filament"
    notes: Custom designed
---

## The idea

A weather station that goes anywhere without needing a power cable. Solar panel keeps the battery topped up, the ESP32 sleeps almost all the time, and once a day it wakes up, reads the sensors, and saves the data.

Pull out your phone, connect to the station's WiFi, and fetch everything it has recorded as a CSV. No cloud, no subscription, no infrastructure. Just a small box on a post that has been quietly watching the weather.

## Why deep sleep matters

Running an ESP32 flat out with WiFi active will drain a LiPo in a few hours. Running it in deep sleep between readings drops the current to microamps. A small solar panel can keep up with that easily — even on a cloudy day.

The plan is to wake once at a set time, take readings, save them, then go back to sleep. WiFi only activates when a phone connects to fetch data. Everything else stays off.

## Storage: LittleFS, not NVS

NVS is shaped for key-value config — settings and calibration values that get written once and read often. It is not designed for accumulating records and hits its ceiling faster than expected with structured data.

LittleFS with a rolling CSV file is the right tool here. More headroom, a cleaner mental model, and you pull a CSV straight to your phone instead of awkwardly enumerating key names. The LittleFS guide in the memory series covers exactly this.

## What it measures

- Temperature, humidity, atmospheric pressure (BME280 - three wires, done in an afternoon)
- Wind speed (reed switch anemometer — already purchased)

The anemometer is a reed switch type: one GPIO interrupt counting magnet pulses over a sample window, multiplied by the manufacturer's calibration constant. Because the ESP32 is asleep most of the time, the approach is to wake up, sample for 30-60 seconds, log the average, then sleep again. Gusts between readings are missed, but that is acceptable for a once-daily station.

## Placement

The whole point is that it has no mains cable. Mount it on a fence post, a roof bracket, a balcony rail — wherever makes sense. The solar panel handles the rest.

## Status

Planning. Anemometer already purchased. Enclosure not designed.
