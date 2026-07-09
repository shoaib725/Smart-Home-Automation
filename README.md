# 🏠 Smart Home Automation System

**Multi-Sensor Integration on Arduino UNO for Security, Comfort & Safety Automation**

![Platform](https://img.shields.io/badge/Platform-Arduino%20UNO-00979D?style=flat-square&logo=arduino)
![Language](https://img.shields.io/badge/Language-C%2B%2B%20(Arduino)-blue?style=flat-square)
![Simulation](https://img.shields.io/badge/Simulated%20on-Tinkercad-orange?style=flat-square)
![Status](https://img.shields.io/badge/Status-Simulation%20Validated-brightgreen?style=flat-square)

---

## 📋 Overview

This project unifies **five independent home-automation functions** into a single, continuously running firmware loop on one Arduino UNO — no RTOS, no second microcontroller. It demonstrates practical multi-sensor fusion on a resource-constrained 8-bit platform, with live status feedback on a 16x2 I2C LCD and the Serial Monitor.

The system was fully designed, wired, and validated in **Tinkercad Circuits** prior to physical prototyping.

### Automated Subsystems

| # | Subsystem | Sensor | Actuator |
|---|-----------|--------|----------|
| 1 | Automatic Door Lock/Unlock | HC-SR04 Ultrasonic | SG90 Servo Motor |
| 2 | Ambient Light Control | LDR | LED |
| 3 | Temperature-Based Fan Speed | TMP36 | DC Fan (NPN transistor, PWM) |
| 4 | Gas Leak Detection | MQ-2 | Piezo Buzzer (2-beep pattern) |
| 5 | Secret Locker Motion Alert | PIR Motion Sensor | Piezo Buzzer (1-beep pattern) |

All five states are also displayed in rotation on a 16x2 I2C LCD (address `0x27`).

---

## ⚙️ Hardware Components

| Component | Function | Pin(s) |
|---|---|---|
| Arduino UNO | Central microcontroller running all subsystem logic | — |
| HC-SR04 Ultrasonic Sensor | Measures distance to detect a person at the door | Trig: D11, Echo: D10 |
| SG90 Micro Servo Motor | Physically locks/unlocks the door mechanism | D5 (PWM) |
| LDR | Senses ambient light level for automatic lighting | A1 |
| LED | Represents the room/porch light | D4 |
| TMP36 Temperature Sensor | Measures ambient temperature for fan control | A0 |
| DC Fan (via NPN transistor) | Cools the room; speed varies with temperature | D9 (PWM) |
| MQ-2 Gas Sensor | Detects LPG/smoke concentration in air | A2 |
| PIR Motion Sensor | Detects unauthorised motion near a secret locker | D6 |
| Piezo Buzzer | Audible alarm — 2 beeps for gas leak, 1 beep for PIR motion | D2 |
| 16x2 I2C LCD Display | Rotates through five status screens for live monitoring | SDA/SCL (I2C, addr 0x27) |

---

## 🧠 Working Principle

**Door Lock/Unlock** — HC-SR04 measures distance via echo timing (`distance = duration × 0.034 / 2`). At ≤ 50 cm, servo rotates to 90° (unlocked); beyond that, it returns to 0° (locked). Servo commands are only issued on state change to avoid redundant writes.

**Ambient Light Automation** — LDR on A1 forms a voltage divider. Reading < 500 → LED ON (dark); ≥ 500 → LED OFF (bright).

**Temperature-Based Fan Speed Control** — TMP36 on A0, converted using a 0.488 °C/ADC-count scale factor. Three PWM tiers via NPN transistor:
- `< 23°C` → OFF
- `23°C – 32.9°C` → Medium (PWM 128)
- `≥ 33°C` → Full speed (PWM 255)

**Gas Leak Detection** — MQ-2 on A2. Reading > 550 → gas leak flagged, distinct 2-beep buzzer alarm.

**Secret Locker Motion Alert** — PIR on D6. HIGH signal → single-beep buzzer alarm, audibly distinct from the gas alarm.

**Status Display** — 16x2 I2C LCD cycles through 5 screens (800 ms each): door status/distance, light status/LDR value, temperature/fan PWM, gas value/leak status, locker motion status — mirrored in detail on the Serial Monitor.

---

## 📊 Threshold Reference

| Parameter | Threshold | Triggered Action |
|---|---|---|
| Door distance | ≤ 50 cm | Servo → 90° (unlocked) |
| Door distance | > 50 cm | Servo → 0° (locked) |
| LDR reading | < 500 | LED ON (dark) |
| LDR reading | ≥ 500 | LED OFF (bright) |
| Temperature | < 23°C | Fan OFF (PWM = 0) |
| Temperature | 23°C – 32.9°C | Fan medium (PWM = 128) |
| Temperature | ≥ 33°C | Fan full speed (PWM = 255) |
| Gas sensor value | > 550 | Gas leak alarm — 2 buzzer beeps |
| PIR state | HIGH | Locker motion alarm — 1 buzzer beep |

---

## 🏗️ Firmware Architecture

The firmware follows a **polling architecture**:

- `setup()` initialises all pins, attaches the servo in the locked position, and boots the I2C LCD.
- `loop()` sequentially calls one handler function per subsystem every 500 ms, followed by the multi-screen LCD refresh routine.

Each subsystem is encapsulated in its own function for modularity:

```
handleDoorLock()
handleAutoLight()
handleFanSpeed()
handleGasSensor()
handlePIRMotion()
```

A shared `soundBeeps(n)` helper is reused by both the gas and PIR handlers — keeping alarm logic DRY while still producing two audibly distinct alert patterns.

**Representative excerpt — door subsystem:**

```cpp
duration = pulseIn(echoPin, HIGH, 30000);
distanceCm = duration * 0.034 / 2;

if (distanceCm > 0 && distanceCm <= DOOR_DISTANCE_THRESHOLD) {
  if (!doorOpen) { doorServo.write(90); doorOpen = true; }
} else {
  if (doorOpen) { doorServo.write(0); doorOpen = false; }
}
```

---

## ✅ Results and Observations

- Door mechanism correctly unlocked within 50 cm and re-locked on object removal, with no false triggering observed.
- LED tracked LDR readings accurately across the 500 threshold.
- Fan PWM stepped cleanly across all three tiers as simulated TMP36 voltage varied.
- Gas and PIR alarms triggered independently via the shared `soundBeeps()` routine without interference.
- LCD and Serial Monitor logs (`[Door]`, `[Light]`, `[Fan]`, `[Gas]`, `[PIR]`) stayed consistent across every loop cycle.

> **Note on responsiveness:** the additive 800 ms delay per LCD screen (4 seconds total per loop) was observed to slow perceived sensor-update responsiveness in Tinkercad — a known trade-off of blocking `delay()` calls across multiple display screens (see Limitations).

---

## 👍 Advantages

- Consolidates five home-automation functions into a single, low-cost Arduino UNO platform.
- Modular handler-per-subsystem structure — easy to read, maintain, and extend.
- Dual feedback channels (LCD + Serial Monitor) for both end-user visibility and developer debugging.
- Distinct buzzer patterns let the occupant audibly distinguish a gas hazard from an intrusion event.
- Fully validated in simulation before any physical components are purchased.

## ⚠️ Limitations

- Relies entirely on `delay()`-based blocking timing — an emergency occurring mid-LCD-cycle can be delayed on-screen by up to ~4 seconds, though the buzzer itself fires immediately.
- Door lock logic is purely distance-based — no identity verification, so it can't distinguish an authorised person from an intruder.
- MQ-2 gas sensor requires real-hardware warm-up/calibration not modelled in Tinkercad.

## 🚀 Future Scope

- Replace blocking `delay()` calls with a `millis()`-based non-blocking scheduler.
- Add RFID or keypad-based authentication at the door.
- Integrate Wi-Fi/Bluetooth (e.g. ESP32/ESP8266) for real-time mobile alerts.
- Log sensor history to an SD card or cloud dashboard for trend analysis.
- Introduce automatic threshold calibration from ambient baseline readings at startup.

---

## 🛠️ Tech Stack

- **Microcontroller:** Arduino UNO
- **Language:** C++ (Arduino)
- **Simulation:** Tinkercad Circuits
- **Libraries:** `Servo`, `Wire`, `LiquidCrystal_I2C`

---

## 👤 Author

**Pathan Mohammad Shoaib Khan**
Embedded Full Stack & IoT Analyst Intern, SRM University AP, Amaravati
B.Tech, Electronics and Communication Engineering (2026)

---

## 📚 References

- Arduino Official Documentation — Servo, Wire, and LiquidCrystal_I2C libraries
- HC-SR04 Ultrasonic Ranging Module — datasheet and application notes
- TMP36 Low Voltage Temperature Sensor — Analog Devices datasheet
- MQ-2 Gas Sensor — Winsen Electronics datasheet
- Tinkercad Circuits Simulation Platform, Autodesk
