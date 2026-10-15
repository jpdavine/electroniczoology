---
slug: daveswf-irrigation-controller
title: DavesWf - Six-Valve WiFi Irrigation Controller
description: An ESP32 controlling six solenoid valves with a phone-friendly web UI, two independent schedules per zone, and automatic fallback to its own hotspot if home WiFi is out of reach.
date: 2026-10-15
category: ESP32
status: Prototype
tags: [ESP32, WiFi, Irrigation, Web UI, Arduino, L298N]
youtube:
parts:
  - name: ESP32 DevKit (38-pin)
    qty: 1
    price: "$8"
    notes: CH340 USB-serial, flashed with arduino-cli
  - name: L298N motor driver board
    qty: 2
    price: "$4 each"
    notes: Each board's 4 outputs used individually as half-bridges, not paired - gives 4 valve channels per board
  - name: FPD-270A 12V solenoid valve (normally closed)
    qty: 6
    price: "$10 each"
  - name: 12V power supply
    qty: 1
    price: "$8"
    notes: Powers the L298N boards and valves, separate from the ESP32's USB power
  - name: LEDs + 220-330 ohm resistors
    qty: 6
    price: "on hand"
    notes: Stand-in for the real valves during bench testing - same GPIO logic
---

## The idea

Six zones, one ESP32, a web page you can reach from your phone. Manual run with a countdown, two independent day/time schedules per valve, settings saved to flash so a power cut does not wipe the config. No app, no cloud account, no subscription - just a browser pointed at an IP address.

## Hardware

Each valve gets its own GPIO, driving one channel of an L298N board used as a plain half-bridge (not the usual paired full H-bridge wiring) straight into an FPD-270A 12V normally-closed solenoid. Two L298N boards cover six channels across GPIOs 25, 26, 27, 32, 33 and 13 - all non-strapping pins, so nothing weird happens at boot. Every pin is forced low before anything else runs, so the valves stay shut through a reset or brownout.

The ESP32's own power stays on USB, separate from the 12V supply feeding the valves and driver boards. Keeping those rails apart means a solenoid under load never has a chance to brown out the microcontroller.

## The web UI

The whole control page is served directly from the ESP32 itself - no external server. It is a single embedded HTML/JS page: each zone gets a card with a manual Run button and countdown, a Stop, and an expandable section for the two schedule slots.

Each schedule slot has its own enable checkbox, a 24-hour HH:MM time, a duration in minutes, and a day-of-week picker. The two slots per valve are fully independent, so a zone can run morning and evening on different days without any extra configuration gymnastics.

If the board cannot join the home network within 15 seconds of boot, it falls back to its own hotspot (`ValveController`) at `192.168.4.1` and keeps retrying the home network quietly in the background. Once it is on the home network it is also reachable at `valves.local` over mDNS, no need to hunt for an IP.

## The code

Two files: `DavesWf.ino` is the logic, `page.h` is the web page, kept separate for the reason explained below. Fill in your own WiFi details at the top before flashing.

```cpp
// DavesWf.ino
/*
 * We stand on the shoulders of giants when we build
 * with knowledge gained from others' efforts.
 * That doesn't make us giants. Be humble.
 * Create with care. Open source is the way.
 *
 * DavesWf - Six-Valve Irrigation Controller
 * ------------------------------------------
 * 6x FPD-270A 12V normally-closed solenoid valves driven through
 * 2x L298N boards (each output used as an independent half-bridge).
 * Serves its own phone-friendly web page: manual run with countdown,
 * plus two independent day/time schedules per valve, 24-hour format.
 * Settings persist to flash. Falls back to its own WiFi hotspot
 * (ValveController / valves1234 at http://192.168.4.1) if it can't
 * join your network, and keeps retrying in the background. Reachable
 * at http://valves.local once connected.
 *
 * Board:   ESP32 Dev Module (Arduino ESP32 core 2.x or 3.x)
 * Library: WiFi, WebServer, ESPmDNS, Preferences - all bundled with
 *          the ESP32 core, no extra installs needed
 *
 * Wiring (ESP32 Dev Module):
 *   GPIO25, 26, 27, 32, 33, 13  ->  one per valve, to an L298N INx pin
 *   (non-strapping pins, all forced LOW before anything else runs)
 *
 * Open source - MIT Licence
 * Electronic Zoology - field notes from the garage
 * https://electroniczoology.com/projects/daveswf-irrigation-controller
 */

#include <WiFi.h>
#include <WebServer.h>
#include <ESPmDNS.h>
#include <Preferences.h>
#include <time.h>
#include <sys/time.h>

// ---------- User settings ----------
const char* WIFI_SSID = "your-ssid";
const char* WIFI_PASS = "your-password";
const char* AP_SSID   = "ValveController";
const char* AP_PASS   = "valves1234";          // 8+ chars
const char* HOSTNAME  = "valves";
const char* TZ_INFO   = "AEST-10AEDT,M10.1.0,M4.1.0/3";  // Victoria, with daylight saving

#define NUM_VALVES   6
#define SLOTS        2      // schedules per valve
#define MAX_RUN_MIN  120    // safety cap on any single run

// Non-strapping GPIOs, safe at boot. One per valve, to L298N INx.
const uint8_t VALVE_PINS[NUM_VALVES] = {25, 26, 27, 32, 33, 13};

// ---------- Data ----------
struct Slot {
  uint8_t  enabled;
  uint8_t  days;      // bit0 = Sunday ... bit6 = Saturday
  uint8_t  hour;
  uint8_t  minute;
  uint16_t duration;  // minutes
};

struct ValveCfg {
  char name[20];
  Slot slot[SLOTS];
};

ValveCfg cfg[NUM_VALVES];

bool     running[NUM_VALVES];
uint32_t runStartMs[NUM_VALVES];
uint32_t runDurMs[NUM_VALVES];

WebServer   server(80);
Preferences prefs;
int lastMinuteKey = -1;

// ---------- Web page ----------
#include "page.h"

// ---------- Valve control ----------
uint32_t remainingMs(int i) {
  if (!running[i]) return 0;
  uint32_t el = millis() - runStartMs[i];
  return el >= runDurMs[i] ? 0 : runDurMs[i] - el;
}

void stopValve(int i) {
  digitalWrite(VALVE_PINS[i], LOW);
  if (running[i]) Serial.printf("Valve %d OFF\n", i + 1);
  running[i] = false;
}

// fromSchedule: a schedule never shortens a longer run already in progress
void startValve(int i, uint16_t minutes, bool fromSchedule) {
  if (minutes < 1) minutes = 1;
  if (minutes > MAX_RUN_MIN) minutes = MAX_RUN_MIN;
  uint32_t dur = (uint32_t)minutes * 60000UL;
  if (fromSchedule && running[i] && remainingMs(i) >= dur) return;
  runStartMs[i] = millis();
  runDurMs[i]   = dur;
  running[i]    = true;
  digitalWrite(VALVE_PINS[i], HIGH);
  Serial.printf("Valve %d ON for %u min (%s)\n", i + 1, minutes, fromSchedule ? "timer" : "manual");
}

// ---------- Storage ----------
void setDefaults() {
  for (int i = 0; i < NUM_VALVES; i++) {
    snprintf(cfg[i].name, sizeof(cfg[i].name), "Zone %d", i + 1);
    for (int s = 0; s < SLOTS; s++) {
      cfg[i].slot[s] = {0, 0x7F, (uint8_t)(6 + s * 12), 0, 10};  // 6:00 and 18:00, off
    }
  }
}

void loadCfg() {
  prefs.begin("valves", true);
  bool ok = prefs.getBytesLength("cfg") == sizeof(cfg);
  if (ok) prefs.getBytes("cfg", cfg, sizeof(cfg));
  prefs.end();
  if (!ok) setDefaults();
}

void saveCfg() {
  prefs.begin("valves", false);
  prefs.putBytes("cfg", cfg, sizeof(cfg));
  prefs.end();
}

// ---------- Time ----------
bool timeValid() { return time(nullptr) > 1700000000; }

void checkSchedules() {
  if (!timeValid()) return;
  time_t t = time(nullptr);
  struct tm tm;
  localtime_r(&t, &tm);
  int key = tm.tm_yday * 1440 + tm.tm_hour * 60 + tm.tm_min;
  if (key == lastMinuteKey) return;
  lastMinuteKey = key;

  for (int i = 0; i < NUM_VALVES; i++) {
    for (int s = 0; s < SLOTS; s++) {
      const Slot& sl = cfg[i].slot[s];
      if (sl.enabled && ((sl.days >> tm.tm_wday) & 1) &&
          sl.hour == tm.tm_hour && sl.minute == tm.tm_min && sl.duration > 0) {
        startValve(i, sl.duration, true);
      }
    }
  }
}

// ---------- Web handlers ----------
String jsonEsc(const char* s) {
  String o;
  for (; *s; s++) {
    char c = *s;
    if (c == '"' || c == '\\') { o += '\\'; o += c; }
    else if ((uint8_t)c >= 0x20) o += c;
  }
  return o;
}

int argValve() {
  if (!server.hasArg("v")) return -1;
  int v = server.arg("v").toInt();
  return (v >= 0 && v < NUM_VALVES) ? v : -1;
}

void handleRoot() { server.send_P(200, "text/html", INDEX_HTML); }

void handleState() {
  time_t t = time(nullptr);
  struct tm tm;
  localtime_r(&t, &tm);
  char hhmm[8];
  snprintf(hhmm, sizeof(hhmm), "%02d:%02d", tm.tm_hour, tm.tm_min);

  String j;
  j.reserve(2200);
  j += "{\"time\":\""; j += hhmm;
  j += "\",\"timeValid\":"; j += timeValid() ? "true" : "false";
  j += ",\"maxRun\":"; j += (int)MAX_RUN_MIN;
  j += ",\"slots\":"; j += (int)SLOTS;
  j += ",\"valves\":[";
  for (int i = 0; i < NUM_VALVES; i++) {
    if (i) j += ',';
    j += "{\"name\":\""; j += jsonEsc(cfg[i].name);
    j += "\",\"on\":"; j += running[i] ? "true" : "false";
    j += ",\"remaining\":"; j += (int)(remainingMs(i) / 1000);
    j += ",\"slots\":[";
    for (int s = 0; s < SLOTS; s++) {
      const Slot& sl = cfg[i].slot[s];
      if (s) j += ',';
      j += "{\"en\":"; j += (int)sl.enabled;
      j += ",\"days\":"; j += (int)sl.days;
      j += ",\"h\":"; j += (int)sl.hour;
      j += ",\"m\":"; j += (int)sl.minute;
      j += ",\"dur\":"; j += (int)sl.duration;
      j += '}';
    }
    j += "]}";
  }
  j += "]}";
  server.send(200, "application/json", j);
}

void handleRun() {
  int v = argValve();
  if (v < 0) { server.send(400, "text/plain", "bad valve"); return; }
  startValve(v, (uint16_t)constrain(server.arg("min").toInt(), 1, MAX_RUN_MIN), false);
  server.send(200, "text/plain", "ok");
}

void handleStop() {
  if (server.arg("v") == "all") {
    for (int i = 0; i < NUM_VALVES; i++) stopValve(i);
  } else {
    int v = argValve();
    if (v < 0) { server.send(400, "text/plain", "bad valve"); return; }
    stopValve(v);
  }
  server.send(200, "text/plain", "ok");
}

void handleConfig() {
  int v = argValve();
  if (v < 0) { server.send(400, "text/plain", "bad valve"); return; }

  if (server.hasArg("name")) {
    String in = server.arg("name");
    char out[20];
    int n = 0;
    for (size_t k = 0; k < in.length() && n < 19; k++) {
      char c = in[k];
      if (isalnum((unsigned char)c) || c == ' ' || c == '-' || c == '_' || c == '.') out[n++] = c;
    }
    out[n] = 0;
    if (n == 0) snprintf(out, sizeof(out), "Zone %d", v + 1);
    strncpy(cfg[v].name, out, sizeof(cfg[v].name));
  }

  for (int s = 0; s < SLOTS; s++) {
    String p = "s" + String(s);
    Slot& sl = cfg[v].slot[s];
    if (server.hasArg(p + "en"))   sl.enabled  = server.arg(p + "en").toInt() ? 1 : 0;
    if (server.hasArg(p + "h"))    sl.hour     = constrain(server.arg(p + "h").toInt(), 0, 23);
    if (server.hasArg(p + "m"))    sl.minute   = constrain(server.arg(p + "m").toInt(), 0, 59);
    if (server.hasArg(p + "dur"))  sl.duration = constrain(server.arg(p + "dur").toInt(), 1, MAX_RUN_MIN);
    if (server.hasArg(p + "days")) sl.days     = server.arg(p + "days").toInt() & 0x7F;
  }
  saveCfg();
  server.send(200, "text/plain", "ok");
}

// Lets the phone set the clock when there's no internet (hotspot mode)
void handleSetTime() {
  long epoch = server.arg("epoch").toInt();
  if (epoch > 1700000000) {
    struct timeval tv = {(time_t)epoch, 0};
    settimeofday(&tv, nullptr);
    Serial.println("Clock set from browser");
  }
  server.send(200, "text/plain", "ok");
}

// ---------- Setup / loop ----------
void setup() {
  // Outputs low before anything else so valves stay shut on boot
  for (int i = 0; i < NUM_VALVES; i++) {
    pinMode(VALVE_PINS[i], OUTPUT);
    digitalWrite(VALVE_PINS[i], LOW);
    running[i] = false;
  }

  Serial.begin(115200);
  delay(200);
  loadCfg();

  WiFi.mode(WIFI_STA);
  WiFi.setHostname(HOSTNAME);
  WiFi.setAutoReconnect(true);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  uint32_t t0 = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - t0 < 15000) delay(250);

  if (WiFi.status() == WL_CONNECTED) {
    Serial.print("Connected. IP: ");
    Serial.println(WiFi.localIP());
  } else {
    WiFi.mode(WIFI_AP_STA);
    WiFi.softAP(AP_SSID, AP_PASS);
    WiFi.begin(WIFI_SSID, WIFI_PASS);   // keep trying in background
    Serial.print("Hotspot mode. IP: ");
    Serial.println(WiFi.softAPIP());
  }

  configTzTime(TZ_INFO, "pool.ntp.org", "time.google.com");
  if (MDNS.begin(HOSTNAME)) MDNS.addService("http", "tcp", 80);

  server.on("/", HTTP_GET, handleRoot);
  server.on("/api/state", HTTP_GET, handleState);
  server.on("/api/run", HTTP_POST, handleRun);
  server.on("/api/stop", HTTP_POST, handleStop);
  server.on("/api/config", HTTP_POST, handleConfig);
  server.on("/api/settime", HTTP_POST, handleSetTime);
  server.onNotFound([]() { server.send(404, "text/plain", "not found"); });
  server.begin();
}

void loop() {
  server.handleClient();
  for (int i = 0; i < NUM_VALVES; i++) {
    if (running[i] && remainingMs(i) == 0) stopValve(i);
  }
  checkSchedules();
  delay(2);
}
```

```cpp
// page.h
#ifndef DAVESWF_PAGE_H
#define DAVESWF_PAGE_H

// Web UI page for DavesWf, kept in its own header.
// Deliberately NOT in the .ino: Arduino's ctags-based prototype generator
// misparses the embedded JS "function"/"async function" definitions as
// real C++ functions and injects garbage forward-declarations + #line
// directives into this string if it lives in the .ino file, corrupting
// the HTML/JS actually served to the browser. .h/.cpp files are not
// scanned by that pass, so keeping it here avoids the bug.

const char INDEX_HTML[] PROGMEM = R"rawliteral(<!DOCTYPE html><html><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Valves</title>
<style>
:root{--bg:#f4f5f2;--card:#fff;--tx:#1d2320;--mut:#6b736e;--on:#1f8a4c;--ac:#2d6cdf;--bd:#dfe3dd}
@media(prefers-color-scheme:dark){:root{--bg:#121513;--card:#1c201d;--tx:#e8ece9;--mut:#9aa39d;--bd:#2e342f}}
*{box-sizing:border-box}body{margin:0;font-family:system-ui,sans-serif;background:var(--bg);color:var(--tx)}
header{display:flex;justify-content:space-between;align-items:center;padding:12px 16px;position:sticky;top:0;background:var(--bg);border-bottom:1px solid var(--bd);z-index:2}
h1{font-size:18px;margin:0}#clk{color:var(--mut);font-size:14px;margin-right:6px}
main{padding:12px 16px;max-width:600px;margin:auto}
.card{background:var(--card);border:1px solid var(--bd);border-radius:12px;padding:14px;margin-bottom:12px}
.top{display:flex;justify-content:space-between;align-items:center}
.nm{font-weight:600;font-size:16px}
.st{font-size:13px;padding:3px 10px;border-radius:99px;background:var(--bd);color:var(--mut)}
.st.on{background:var(--on);color:#fff}
.row{display:flex;gap:8px;margin-top:10px;align-items:center;flex-wrap:wrap}
input,button{font:inherit;padding:9px 10px;border-radius:8px;border:1px solid var(--bd);background:var(--bg);color:var(--tx)}
input[type=number]{width:76px}
button{cursor:pointer}.go{background:var(--on);color:#fff;border:0}.stop{background:#c0392b;color:#fff;border:0}.pri{background:var(--ac);color:#fff;border:0}
details{margin-top:10px}summary{color:var(--ac);cursor:pointer}
.slot{border-top:1px solid var(--bd);padding-top:6px;margin-top:10px}
.days{gap:4px}.days label{width:36px;text-align:center;padding:7px 0;border:1px solid var(--bd);border-radius:6px;font-size:13px;user-select:none;cursor:pointer}
.days input{display:none}.days label.sel{background:var(--ac);color:#fff;border-color:var(--ac)}
.warn{background:#c98a00;color:#fff;padding:8px 12px;border-radius:8px;margin-bottom:12px;display:none}
small{color:var(--mut)}
</style></head><body>
<header><h1>Valves</h1><div><span id="clk">--:--</span><button class="stop" onclick="stopAll()">Stop all</button></div></header>
<main><div class="warn" id="warn">Clock not set. Syncing from this phone.</div><div id="v"></div></main>
<script>
let S=null;const DN=['S','M','T','W','T','F','S'];
const $=id=>document.getElementById(id);
const esc=s=>String(s).replace(/[&<>"]/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
const p2=n=>String(n).padStart(2,'0');
// 24h HH:MM auto-format as the user types digits into a timer field
function timeFmt(el){let d=el.value.replace(/\D/g,'').slice(0,4);
if(d.length>=3)d=d.slice(0,2)+':'+d.slice(2);
if(d.length>=2){let h=+d.slice(0,2);if(h>23)d='23'+d.slice(2)}
if(d.length===5){let m=+d.slice(3);if(m>59)d=d.slice(0,3)+'59'}
el.value=d}
const fmt=t=>Math.floor(t/60)+':'+p2(t%60);
async function post(u){try{await fetch(u,{method:'POST'})}catch(e){}load(false)}
async function load(full){try{const r=await fetch('/api/state');S=await r.json();if(full||!$('c0'))build();update()}catch(e){$('clk').textContent='offline'}}
function build(){$('v').innerHTML=S.valves.map((v,i)=>`
<div class="card" id="c${i}"><div class="top"><span class="nm" id="n${i}">${esc(v.name)}</span><span class="st" id="s${i}">Off</span></div>
<div class="row"><input type="number" id="m${i}" min="1" max="${S.maxRun}" value="10"><small>min</small>
<button class="go" onclick="run(${i})">Run</button><button class="stop" onclick="post('/api/stop?v=${i}')">Stop</button></div>
<details><summary>Timers and name</summary>
<div class="row"><input id="nm${i}" maxlength="19" value="${esc(v.name)}" style="flex:1"></div>
${v.slots.map((s,j)=>`<div class="slot"><div class="row"><label><input type="checkbox" id="e${i}_${j}" ${s.en?'checked':''}> Timer ${j+1}</label>
<input type="text" inputmode="numeric" pattern="([01][0-9]|2[0-3]):[0-5][0-9]" placeholder="HH:MM" maxlength="5" style="width:64px" id="t${i}_${j}" value="${p2(s.h)}:${p2(s.m)}" oninput="timeFmt(this)"><input type="number" id="d${i}_${j}" min="1" max="${S.maxRun}" value="${s.dur}"><small>min</small></div>
<div class="row days">${DN.map((d,k)=>`<label class="${(s.days>>k)&1?'sel':''}"><input type="checkbox" id="w${i}_${j}_${k}" ${(s.days>>k)&1?'checked':''} onchange="this.parentNode.classList.toggle('sel',this.checked)">${d}</label>`).join('')}</div></div>`).join('')}
<div class="row"><button class="pri" onclick="save(${i})">Save</button><small id="ok${i}"></small></div></details></div>`).join('')}
function update(){$('clk').textContent=S.timeValid?S.time:'no clock';$('warn').style.display=S.timeValid?'none':'block';
if(!S.timeValid&&!window.synced){window.synced=1;post('/api/settime?epoch='+Math.floor(Date.now()/1000))}
S.valves.forEach((v,i)=>{const s=$('s'+i);s.className='st'+(v.on?' on':'');s.textContent=v.on?'On '+fmt(v.remaining):'Off';$('n'+i).textContent=v.name})}
function run(i){post(`/api/run?v=${i}&min=${$('m'+i).value}`)}
function stopAll(){post('/api/stop?v=all')}
async function save(i){let q=`/api/config?v=${i}&name=${encodeURIComponent($('nm'+i).value)}`;
for(let j=0;j<S.slots;j++){const t=$(`t${i}_${j}`).value.split(':');let days=0;
for(let k=0;k<7;k++)if($(`w${i}_${j}_${k}`).checked)days|=1<<k;
q+=`&s${j}en=${$(`e${i}_${j}`).checked?1:0}&s${j}h=${+t[0]||0}&s${j}m=${+t[1]||0}&s${j}dur=${$(`d${i}_${j}`).value}&s${j}days=${days}`}
let ok=false;try{ok=(await fetch(q,{method:'POST'})).ok}catch(e){}
$('ok'+i).textContent=ok?'Saved':'Error';setTimeout(()=>$('ok'+i).textContent='',2000);load(false)}
load(true);setInterval(()=>load(false),2000);
</script></body></html>)rawliteral";

#endif
```

## A bug worth writing down

The first flash looked fine from the terminal - compiled clean, uploaded clean, server answered on curl. But the page loaded with an empty valve list. No cards, nothing below the header.

The API was returning good data. The problem turned out to be the Arduino build process itself: the page's HTML and JavaScript lived in a big raw string literal directly inside the `.ino` file. Arduino's automatic function-prototype generator scans the whole sketch for anything that looks like a C++ function definition - and it misread the embedded JS `function build(){` and `async function post(u){` as real C++ functions. It then injected garbled forward-declarations and `#line` directives straight into the string's own bytes, corrupting exactly what got served to the browser and throwing a hard JavaScript syntax error before any of the page's logic could run.

The fix was to move the HTML/JS out of the `.ino` entirely and into its own header file. Arduino's prototype scanner only touches `.ino` files, not `.h` files sitting in the same sketch folder, so splitting it out sidesteps the bug completely. Lesson for next time: any sketch serving a sizeable embedded web page belongs in its own header from the start.

## Status

Firmware is built, flashed, and running on real hardware - connected to the home network, web UI fully working, schedules saving to flash correctly. Nothing physical is wired to the valve outputs yet. Next step is bench-testing with LEDs standing in for the solenoids (same GPIO high/low logic, just lower stakes while confirming the schedules behave), then wiring up the real L298N boards and valves.
