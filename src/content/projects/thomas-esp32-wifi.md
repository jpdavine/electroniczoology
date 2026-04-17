---
slug: thomas-esp32-wifi
title: Thomas the Tank Engine — WiFi-Enabled Rolling Experiment
description: Gutting a toy Thomas the Tank Engine and turning it into a WiFi-controlled embedded systems platform with an ESP32-C3. Because why not.
date: 2026-04-17
category: ESP32
tags: [ESP32-C3, WiFi, Arduino, Motor Control]
youtube:
parts:
  - name: ESP32-C3 dev board
    qty: 1
    price: "$6"
    link: ""
    notes: Fits inside the Thomas shell
  - name: Thomas the Tank Engine toy
    qty: 1
    price: "e-waste"
    notes: The donor vehicle
  - name: Small DC motors (salvaged)
    qty: 2
    price: "-"
    notes: From the original toy or salvage
  - name: Motor driver module (DRV8833 or similar)
    qty: 1
    price: "$4"
    link: ""
  - name: LiPo battery (small, 3.7V)
    qty: 1
    price: "$8"
    link: ""
  - name: LiPo charging module (TP4056)
    qty: 1
    price: "$2"
    link: ""
  - name: Jumper wires + heatshrink
    qty: 1
    price: "$3"
    link: ""
---

## The idea

Every maker gets to a point where they look at a toy train and think "this should have WiFi." Thomas the Tank Engine — specifically the old chunky plastic kind you find at op shops and garage sales — turns out to be a surprisingly good enclosure for a small ESP32-C3 build.

The goal: pull out whatever electronics are inside, fit an ESP32-C3, wire up motor control, and drive it over WiFi from a phone browser. No app. No custom firmware tool. Just a web page served from the chip itself.

## What's inside a Thomas

Crack one open and you typically find a small DC motor driving the rear wheels, a AA battery holder, and a basic on/off switch. The shell has more space than you'd expect — enough for an ESP32-C3, a small motor driver, and a LiPo if you're tidy about it.

## Wiring overview

The ESP32-C3 runs the motors through a DRV8833 dual H-bridge driver. Two GPIO pins per motor channel give you forward, reverse, and stop. The LiPo feeds through a TP4056 charging module so you can top it up via USB without pulling it apart.

```
LiPo (+) → TP4056 BAT+ → ESP32-C3 5V (via boost) + DRV8833 VM
LiPo (-) → GND (common)
ESP32-C3 GPIO2 → DRV8833 AIN1
ESP32-C3 GPIO3 → DRV8833 AIN2
DRV8833 AOUT1/2 → Motor
```

## The sketch

The ESP32-C3 hosts a tiny web server. Connect your phone to the Thomas WiFi network, open a browser, and you get a control page with forward/reverse/stop buttons.

```cpp
#include <WiFi.h>
#include <WebServer.h>

const char* ssid     = "Thomas";
const char* password = "chugchug";

const int AIN1 = 2;
const int AIN2 = 3;

WebServer server(80);

const char* PAGE = R"(
<html><body style="text-align:center;font-family:sans-serif;padding:2rem">
<h2>Thomas Control</h2>
<a href="/fwd"><button style="padding:1rem 2rem;margin:0.5rem">Forward</button></a>
<a href="/rev"><button style="padding:1rem 2rem;margin:0.5rem">Reverse</button></a>
<a href="/stop"><button style="padding:1rem 2rem;margin:0.5rem">Stop</button></a>
</body></html>
)";

void setup() {
  pinMode(AIN1, OUTPUT);
  pinMode(AIN2, OUTPUT);

  WiFi.softAP(ssid, password);

  server.on("/",    [](){ server.send(200, "text/html", PAGE); });
  server.on("/fwd", [](){ digitalWrite(AIN1,HIGH); digitalWrite(AIN2,LOW);  server.sendHeader("Location","/"); server.send(303); });
  server.on("/rev", [](){ digitalWrite(AIN1,LOW);  digitalWrite(AIN2,HIGH); server.sendHeader("Location","/"); server.send(303); });
  server.on("/stop",[](){ digitalWrite(AIN1,LOW);  digitalWrite(AIN2,LOW);  server.sendHeader("Location","/"); server.send(303); });

  server.begin();
}

void loop() {
  server.handleClient();
}
```

## What I learned

The ESP32-C3 runs hot when the WiFi radio is active and the motor driver is working at the same time — worth keeping an eye on temperatures if it's enclosed tightly. The LiPo charging circuit also needs proper airflow or the TP4056 gets warm.

Fitting everything inside the shell is 80% of the job. The electronics are easy. The origami is hard.
