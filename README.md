# Zelena Skelia EcoProcessor — Pump Controller (ESPHome)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

> ⚠️ Non-commercial, unofficial "as is" project — no warranty, no
> liability on the author's part for any consequences of use. Details:
> section "8. License and Disclaimer" below.

*(Українську версію цього README дивіться в [`README_uk.md`](./README_uk.md).)*

A circulation-pump controller for a biofilter, based on ESP8266 (Wemos
D1 mini), with smooth pulsed power control via an AC dimmer (leading-edge
phase-cut). This firmware is a copy / analog of the **EcoProcessor**
controller firmware and is fully compatible in principle of operation
with **«Zelena Skelia»** biological septic systems — the same cyclic
irrigation algorithm (ramp up → run → ramp down → rest), multi-stage
pump overload protection, and automatic restart after a fault.

---

## Why this controller is needed

> 📺 **Source.** The description of the EcoProcessor's principle of
> operation and its programs (P1/P2/P3, irrigation cycles, energy
> savings) was obtained from an open source — an official video on the
> «Zelena Skelia» YouTube channel:
> https://www.youtube.com/watch?v=CfAyo-JgoCA

### The «Air Ring»® / «Eco Ring» biofilter

Plastic discs **«Air Ring»®** (or **«Eco Ring»**) are the most important
element of a standalone-sewage biofilter. Made of rigid plastic, with a
usable loading surface area of **180 m² per 1 m³**.

- **Biofilm formation.** The discs' main job is to provide an optimal
  environment for beneficial bacteria — naturally present in wastewater —
  to attach and develop. Microorganisms fix themselves onto the plastic
  surface, forming a biofilm, and feed on organic contaminants in the
  wastewater, oxidizing and breaking them down.
- **Self-cleaning ability.** The discs have a unique ability to clean
  themselves of dead biofilm residue. In older generations of filters,
  it was precisely dead bacteria that caused rapid clogging of the media,
  worsened aeration, and putrefaction, which would completely stop the
  treatment process. «Air Ring»® / «Eco Ring» discs require no special
  cleaning or replacement — they are durable and do not degrade during
  operation.
- **Advantages over brush and honeycomb media:** efficiency (greater
  contact between air, water, and microorganisms), environmental
  friendliness (no fine particles released into water or soil),
  cost-effectiveness (allows air flow to be dispersed at lower pressure
  without compressors that must run continuously), practicality
  (successfully tested in over 40,000 biofilters in more than 30
  countries worldwide — from hot climates to the Arctic).

### The role of the circulation pump

1. **Intake of clarified water.** First, domestic wastewater undergoes
   thorough mechanical treatment to remove fats, insoluble matter, and
   solid debris across three sections of the septic tank.
2. **Lifting and irrigation (aeration).** The pump lifts the already
   pre-clarified water up into the biological reactor. There, it
   irrigates (sprays) the liquid over the «Air Ring»® discs several
   times. This saturates the water with oxygen and brings it into active
   contact with the biofilm on the discs, where bacteria complete the
   biological breakdown of organic matter.

Reliable pump equipment from well-known global brands is used for
this — namely **Wilo, Kärcher, or Grundfos**.

> ⚙️ **Max Power and Min Power must be calibrated manually for each
> specific septic tank.** There is no one-size-fits-all setting —
> nozzles/sprayers, lift height, and the number of «Air Ring»® discs
> differ from installation to installation. So after mounting, these two
> parameters need to be **set empirically**, by observing the actual
> irrigation inside the bioreactor: the liquid must **evenly cover and
> irrigate all the discs**, without dry spots (too little power — doesn't
> reach/cover everything) and without overshoot or spraying past the
> media (too much power). Ramp Up/Down then smoothly "swings" the flow
> between these two verified limits, rather than between arbitrary
> values.

### The role of the controller

Thanks to the intelligent **EcoProcessor** control unit, the pump is
protected from overloads through a **multi-stage protection** system.
In addition, the controller optimizes the pump's operating cycles,
reducing the system's overall energy consumption by **more than 10x**.
For example, a Wilo-brand pump in such a system consumes only **33 W of
electricity per hour**.

This ESPHome firmware implements the same idea in an open way:

- **Cyclic irrigation instead of continuous operation** (P1/P2 modes,
  run ⇄ rest) — exactly the cycle optimization that delivers the
  multi-fold energy savings.
- **Smooth phase-cut power ramp up/down** (Ramp Up/Down) instead of a
  sudden full-power switch-on — a gentler start and stop on every
  irrigation cycle, less strain on the pump.
- **Pump current monitoring and automatic restart** — the controller
  watches the current and reacts to abnormal operating conditions
  (overload, rotor jam) by shutting the pump off and trying to restart
  it after a pause; the owner sees the retry counter (`Restart Attempts`)
  in the web UI.

---

## 1. Components

| # | Component | Purpose | Purchase link |
|---|-----------|---------|-----------------|
| 1 | **Wemos D1 mini V4 (ESP8266)** | microcontroller, Wi-Fi, web UI | https://www.aliexpress.com/item/32831353752.html |
| 2 | **AC Dimmer RobotDyn (1-channel)** | leading-edge phase-cut for smooth pump power control | https://www.aliexpress.com/item/32802025086.html |
| 3 | **ACS712 5A current sensor** *(inferred from the 0.185 V/A coefficient in the code — the standard sensitivity of the ACS712-5A, 185 mV/A)* | measures pump current for status indication and fault detection (power terminals IP+/IP− — on the dimmer's output, before the pump) | https://www.aliexpress.com/item/1005008375334741.html |
| 4 | **HLK-5M05 power supply**, isolated AC-DC 220V→5V | powers the Wemos D1 mini and the ACS712's logic side | https://www.aliexpress.com/item/1005009246796870.html |
| 5 | Drainage/circulation pump (AC, single-phase, ~220V) | actuator | depends on the installation, not included |

---

## 2. Wiring diagram (ASCII)

### 2.1 220V power circuit (mains → dimmer → current sensor → pump)

The ACS712 sits **on the dimmer's output** (after the phase cut), in
series with the phase conductor between the dimmer and the pump — i.e.
it measures exactly the current actually going to the pump, not the
overall mains current upstream of the dimmer.

```
  ~220V AC MAINS
   L ●───────────────────────────────────────┐
   N ●───────────────────┐                   │
                          │                   │
                          ▼                   ▼
              ┌─────────────────────────────────────────┐
              │            AC Dimmer RobotDyn             │
              │                                           │
              │   AC-IN(N)                    AC-IN(L)    │
              │                                           │
              │            (triac, phase-cut)             │
              │                                           │
              │   AC-OUT(N)                  AC-OUT(L)    │
              └───────┬───────────────────────────┬───────┘
                      │                            │
                 GATE ●│                  Z-CROSS ●│   ── signals to ESP8266 (see 2.2)
                      │                            │
                      │                            ▼
                      │                  ┌───────────────────┐
                      │                  │     ACS712-5A       │
                      │                  │   IP+ ────── IP−    │  ── current sensor,
                      │                  └──────┬───────┬──────┘     IN SERIES WITH L,
                      │                     OUT │(analog)             AFTER the dimmer
                      │                         │       │
                      ▼                         │       ▼
              ┌───────────────────────────────────────────┐
              │                  PUMP                       │
              │            N                    L           │
              └───────────────────────────────────────────┘
```

### 2.2 Control signals → Wemos D1 mini (ESP8266)

```
  GATE (from dimmer)     ───────────►  GPIO4 (D2)
  Z-CROSS (from dimmer)  ◄───────────  GPIO5 (D1)
  ACS712 OUT (analog)    ───────────►  A0
  LED indicator          ◄───────────  GPIO2 (D4, active LOW)
```

### 2.3 Logic power supply (220V → 5V, isolated)

The HLK-5M05 supplies 5V/GND to **two** consumers at once: the Wemos D1
mini (via the VBUS/5V pin) and the ACS712's logic side (the module's own
VCC/GND pins — not to be confused with the IP+/IP− power terminals,
which sit separately in series with L, see 2.1):

```
                               ┌──► 5V/GND ──► Wemos D1 mini (VBUS, GND)
~220V AC MAINS ──► HLK-5M05 ──┤
   (AC-DC, isolated)           └──► 5V/GND ──► ACS712 (VCC, GND — logic side)
```

**ESP8266 pin map (per the `pump_controller.yaml` code):**

| Wemos pin | Function | Connects to |
|-----------|----------|-------------|
| `GPIO4` (D2) | dimmer `gate_pin` | DIM on the AC Dimmer RobotDyn |
| `GPIO5` (D1) | `zero_cross_pin` (input) | Z-C on the AC Dimmer RobotDyn |
| `GPIO2` (D4) | LED output (inverted — active LOW) | status LED |
| `A0` | ADC analog input | ACS712 current sensor OUT (logic side) |
| `3V3` | dimmer logic power | VCC on the AC Dimmer RobotDyn |
| `5V`, `GND` (VBUS) | power | HLK-5M05 output (shared with ACS712's VCC/GND) |

> ⚠️ **Electric shock hazard.** The AC Dimmer, ACS712 (in series with the
> phase), and HLK-5M05 operate at mains voltage (220V). All wiring work
> must only be done with the panel/line fully de-energized, ideally
> through a breaker and RCD. The Wemos D1 mini's logic side (GPIO/A0/5V)
> is galvanically isolated from the mains only thanks to the isolated
> HLK-5M05 power supply — do not use non-isolated power supplies.

---

## 3. Principle of operation

The firmware implements a non-blocking state machine (`interval:` every
100ms) with four states:

```
      0 = stopped
        │ (Pump: Enabled = ON)
        ▼
      1 = running ──(work phase reached Work Time)──► 2 = resting
        ▲                                                  │
        │◄────────────────(Rest Time elapsed)──────────────┘
        │
        │ (fault: no current while the dimmer is active)
        ▼
      3 = fault ──(Retry Delay elapsed)──► back to 1 = running
```

### Operating modes (`select: Operating Mode`)

Switched via a single selector in the web UI ("Pump Control" group),
no reflashing needed. Changing the mode on the fly immediately starts a
new cycle from the "running" phase (`g_state = 1`, ramp up).

#### P1 Standard — normal operating mode

- Work: **120s (2 min)**, rest: **480s (8 min)** → full cycle 10 min,
  the pump is active ≈ **20%** of the time.
- Use case: typical daily load — wastewater comes in regularly, requires
  regular circulation/discharge without excessive pump wear.
- All 4 timing parameters (Work/Rest/Ramp Up/Ramp Down Time) are
  editable separately in the "Mode P1 — Standard" group.

#### P2 Economy — economy mode

- Work: **60s (1 min)**, rest: **1740s (29 min)** → full cycle 30 min,
  the pump is active only ≈ **3.3%** of the time.
- Use case: periods of low load (night, no occupants, off-season) —
  significantly less pump run-time and power consumption, while the
  liquid still doesn't stagnate.
- Parameters are fully independent from P1 (separate "Mode P2 —
  Economy" group) — both profiles can be kept ready and switched
  between without recalculating anything.

#### P3 Constant — pump testing mode

- The pump runs at **100% power continuously**, with no "run/rest"
  cycling and no level pulsing at all — ramps (Ramp Up/Down) are not
  used here, the dimmer is effectively fully open.
- Use case: **exclusively for testing the pump at full power without
  the dimmer** — to check whether the pump itself is functional and
  draws normal current/flow, without any influence from phase control.
  This is not a normal operating mode for the biofilter.
- Fault detection still works here too, but via a separate code branch:
  it checks directly whether there is current at 100% power, with no
  dependency on the ramp up/down phase (which doesn't exist in this
  mode).

#### Power pulsing within the "run" phase (P1/P2)

Unlike simply "switch on at a fixed power," in P1/P2 modes the power
level within the run phase continuously **pulses in a triangle wave**
between **Min Power** and **Max Power** until the work time ends:

```
level (%)
 Max ─┤      ╱╲          ╱╲          ╱╲
      │     ╱  ╲        ╱  ╲        ╱  ╲
      │    ╱    ╲      ╱    ╲      ╱    ╲
 Min ─┤   ╱      ╲    ╱      ╲    ╱      ╲
      └──┴────────┴──┴────────┴──┴────────┴──► time
         Ramp Up  Ramp Down  Ramp Up  Ramp Down …
         (e.g. 0.1s)  (e.g. 14s)
```

In other words, one "tooth" = Ramp Up Time (fast rise to Max) + Ramp
Down Time (slower decay to Min), repeating in a loop until the run phase
ends. By default the ramp-up is much faster than the ramp-down (0.1s vs
14s) — an almost instant "kick" followed by a smooth fade, rather than a
symmetric sine wave. This is the same "smooth pulsing" mentioned at the
top of this README — it's gentler on the pump and pipes than holding one
fixed power level for the entire run phase.

#### Example factory settings (from the video)

In the official «Zelena Skelia» video (source — see the top of this
README), the manufacturer gives an example configuration of the
EcoProcessor control unit for three septic tank models. The original
notation (as spoken in the video) and its decoding into this firmware's
terms are shown side by side, for verification:

**Video notation:** `H` = Rest (minutes), `H.` = Work (minutes),
`h` = Ramp Down (seconds), `h.` = Ramp Up (seconds), `V` = Min Power (%),
`V.` = Max Power (%).

| Model | Mode | Original from the video | Work Time | Rest Time | Ramp Up | Ramp Down | Min Power | Max Power |
|-------|------|---------------------------|-----------|-----------|---------|-----------|-----------|-----------|
| **ZS4** | P1 (Р-1) | `H8 H.2 \ h14 h.1 \ V32 V.38` | 120s (2 min) | 480s (8 min) | 0.1s | 14s | 32% | 38% |
| **ZS4** | P2 (Р-2) | `H29 H.1 \ h14 h.1 \ V32 V.38` | 60s (1 min) | 1740s (29 min) | 0.1s | 14s | 32% | 38% |
| **ZS6** *(3 turns)* | P1 (Р-1) | `H13 H.2 \ h14 h.1 \ V37 V.55` | 120s (2 min) | 780s (13 min) | 0.1s | 14s | 37% | 55% |
| **ZS6** | P2 (Р-2) | `H29 H.1 \ h14 h.1 \ V37 V.55` | 60s (1 min) | 1740s (29 min) | 0.1s | 14s | 37% | 55% |
| **ZS8** | P1 (Р-1) | `H10 H.2 \ h14 h.1 \ V37 V.55` | 120s (2 min) | 600s (10 min) | 0.1s | 14s | 37% | 55% |
| **ZS8** | P2 (Р-2) | `H29 H.1 \ h14 h.1 \ V37 V.55` | 60s (1 min) | 1740s (29 min) | 0.1s | 14s | 37% | 55% |

Work Time, Rest Time, Ramp Up, and Ramp Down in this firmware's defaults
already match the **ZS4** row in the table above (120s/480s for P1,
60s/1740s for P2, 0.1s/14s ramp up/down). **The default Max Power and
Min Power (75%/55%) do not match the video** — this is either a
different model, or values manually tuned for a specific pump (see the
⚙️ note above about calibrating Max/Min Power for actual disc
irrigation).

### Fault protection

If the dimmer is active (`level ≥ 5%`) and the current sensor (`ACS712`)
sees no current for more than **Fault: Detection Time** seconds — the
system treats this as a fault (overload / rotor jam), shuts the pump
off, waits **Fault: Retry Delay** seconds, and tries to start again. The
retry counter (`Restart Attempts`) is visible in the web UI. This
function can be disabled via the **Fault Detection: Enabled** switch.

### Current measurement (RMS)

Because of the dimmer's phase-cut, the pump current is not a smooth sine
wave but a series of short pulses. `Pump Current` is therefore computed
as a true RMS over ~200 direct ADC samples across ~40ms (≈2 mains
cycles), rather than a single "instantaneous" snapshot — this eliminates
the reading jumps typical of naive single-sample measurement under
phase control.

### LED indication (LED, GPIO2)

A single onboard LED shows the system's current state without needing
to open the web UI. The "tick" logic (`g_led_tick`) generates a cycle of
20 ticks at 100ms each = **2 seconds per full pattern cycle**, and all
the patterns below are built on that cycle length.

| State | What the LED shows | Pattern (1 cell = 100ms, 2s total) |
|-------|----------------------|--------------------------------------|
| **0 — Stopped** | Off continuously | `░░░░░░░░░░░░░░░░░░░░` |
| **1 — Running** | Solid on (no blinking) | `████████████████████` |
| **2 — Resting** | 1 short blink every 2s (100ms on / 1900ms off) | `█░░░░░░░░░░░░░░░░░░░` |
| **3 — Fault** | Double blink every 2s (2×200ms on with a gap, then a long pause) | `██░░██░░░░░░░░░░░░░░` |

*(`█` = LED on, `░` = off; read left to right as 20 consecutive 100ms
segments)*

So, just by looking at the indicator, you can tell the state at a
glance:
- **not blinking, not lit** → pump stopped (Pump: Enabled = OFF, or P3
  hasn't started yet);
- **solidly lit, no blinking** → currently in the "running" phase;
- **one short "blip" every ~2s** → the "rest" phase between cycles
  (Resting) — everything is normal, just waiting;
- **two short blinks in a row every ~2s** → fault, the system is
  waiting before retrying — worth checking the pump/pipe if this
  repeats (see the `Restart Attempts` counter).

> Note: in **P3 Constant** mode, the LED behaves as in the Running
> state (solid on) until a fault occurs — there is no separate pattern
> for P3, since for the controller it's just another kind of "running."

---

## 4. Web interface (`web_server`, port 80)

Settings are grouped on the device's page:

| Group | Contains |
|-------|----------|
| **Pump Control** | operating mode (P1/P2/P3), pump on/off |
| **Power Settings** | Max Power / Min Power (%) — shared between P1 and P2, calibrated empirically for the specific septic tank (see "The role of the circulation pump" above) |
| **Status & Monitoring** | current phase, time remaining (M:SS), power, current, running status |
| **Mode P1 — Standard** | work/rest/ramp-up/ramp-down time + a text minute-hint under each parameter |
| **Mode P2 — Economy** | the same for the economy mode |
| **Fault & Recovery** | fault detection time, retry delay, retry counter |

---

## 5. Installation / flashing

1. Install ESPHome:
   ```bash
   pip install esphome
   ```
2. Open `pump_controller.yaml` and replace the placeholders with your
   actual values (Wi-Fi, API key, OTA password, fallback AP password):

   | Placeholder in the file | What to substitute |
   |---------------------------|-----------------------|
   | `YOUR_WIFI_SSID` / `YOUR_WIFI_PASSWORD` | your main Wi-Fi network |
   | `YOUR_WIFI_SSID_2` / `YOUR_WIFI_PASSWORD_2` | backup Wi-Fi network |
   | `YOUR_API_ENCRYPTION_KEY_BASE64` | Home Assistant API encryption key — generate with: `python3 -c "import base64,os; print(base64.b64encode(os.urandom(32)).decode())"` |
   | `YOUR_OTA_PASSWORD` | your own OTA update password |
   | `YOUR_AP_FALLBACK_PASSWORD` | fallback Access Point password (appears if Wi-Fi is unavailable) |

   > The file is self-contained — no separate `secrets.yaml` is needed,
   > but for that exact reason **do not publish it publicly once you've
   > filled in real values** (it then contains real passwords/keys, not
   > placeholders).
3. First flash — via USB (requires a cable to the Wemos D1 mini):
   ```bash
   esphome run pump_controller.yaml
   ```
4. All subsequent updates — over the network (OTA), no cable needed:
   ```bash
   esphome upload pump_controller.yaml
   ```
5. Device web UI: `http://<device-ip>/` (the config defaults to a
   static address of `192.168.5.123` — change `use_address` under
   `wifi:` if needed).

---

## 6. Safety

- Once you've filled in the placeholders with real Wi-Fi passwords, an
  API key, and an OTA password, **do not publish or commit this
  `pump_controller.yaml` to a public git repo/cloud** — unlike the
  placeholder template, the filled-in file now contains real secrets.
- All 220V wiring work must be done only by a qualified professional
  with the power off, through a circuit breaker and RCD.

---

## 7. Enclosure (3D printing)

The [`3d-print/`](./3d-print/) folder contains an example enclosure for
3D printing:

| File | Purpose |
|------|---------|
| `enclosure-top.stl` | top part of the enclosure |
| `enclosure-bottom.stl` | bottom part of the enclosure |
| `enclosure-latch.stl` | latch/fastener |

> ⚠️ This is a **working "as is" example**, not a refined engineering
> model. The components inside (Wemos, dimmer, ACS712, HLK) were
> attached with hot glue, without precise dimensioning of the mounting
> points — so the enclosure will likely need rework for your specific
> set of components. If you have 3D-design skills, feel free to improve
> the mounts, ventilation, cable entries, etc., and make it better.

---

## 8. License and Disclaimer

**License:** the code in this project (`pump_controller.yaml`) is
distributed under the **MIT License** — see the [`LICENSE`](./LICENSE)
file. You are free to use, copy, modify, and distribute it, with no
warranty.

**Non-commercial, unofficial project.** This is a hobbyist project
built for personal home use. The project is **not affiliated or
associated** with the «EcoProcessor» manufacturer, the **«Zelena
Skelia»** brand, the owners of the «Air Ring»® / «Eco Ring» trademarks,
or the pump manufacturers (Wilo, Kärcher, Grundfos), and is not their
official or licensed firmware or product. All mentioned names are the
property of their respective rights holders, referenced here solely to
explain compatibility and the principle of operation.

**Patents and underlying technology.** The original systems mentioned
in this README (biofilters, pump control algorithms, etc.) may be
protected by patents or other intellectual property rights of their
manufacturers. The description of the EcoProcessor's principle of
operation and its programs in this README is based solely on an **open
source** — an official video on the «Zelena Skelia» YouTube channel
(https://www.youtube.com/watch?v=CfAyo-JgoCA), without access to any
internal/proprietary manufacturer documentation. This project makes no
claim to rights over such technology — the implementation in
`pump_controller.yaml` is an independent, self-made development by the
author for their own equipment.

**Disclaimer.** The firmware and documentation are provided **"AS IS"**,
without any warranty. You install and use this controller **entirely at
your own risk**. The work involves wiring under mains voltage (220V —
electric shock hazard) and controlling the pump of a wastewater
treatment system (risk of equipment damage, overflow, or septic system
failure from incorrect configuration or malfunction). The author
**bears no liability** for any damages, injuries, property damage,
failure of the biological treatment process, or other consequences,
direct or indirect, related to using, assembling, or modifying this
project. Installing the 220V power section is recommended to be
entrusted to a qualified electrician.

---

## 9. Project files

| File | Purpose |
|------|---------|
| `pump_controller.yaml` | main ESPHome configuration (with placeholders instead of secrets) |
| `3d-print/enclosure-top.stl` | top part of the enclosure (3D printing) |
| `3d-print/enclosure-bottom.stl` | bottom part of the enclosure (3D printing) |
| `3d-print/enclosure-latch.stl` | enclosure latch/fastener (3D printing) |
| `LICENSE` | MIT license text |
| `README.md` | this file (English) |
| `README_uk.md` | Ukrainian version of this README |
