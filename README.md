# Genny Monitor (ESP32) — README

A small ESP32-based monitor/controller that:

* Reads **battery voltage**, **temperature**, and **humidity**.
* Shows a clean **OLED UI** (SSD1306).
* Uses **non-blocking Wi-Fi** with **30 s** retries.
* **Never blocks** on Wi-Fi: when offline it **only displays V/T/H** and disables network actions.
* Includes a **built-in captive portal** (no extra libraries) to configure:

  * Wi-Fi SSID/Password
  * `gennyName`, `deviceName`, `batteryDeviceName`
  * Admin username/password (for portal)
  * `voltageCalFactor` (calibration)
* Persists settings in **NVS/Preferences**.
* Talks to your backend:

  * `GET /getGeneratorState?gennyName=…`
  * `POST /setGeneratorState`
  * `POST /setSensorReading`

> The code is self-contained (core ESP32 libs only): `WiFi`, `WebServer`, `DNSServer`, `Preferences`, plus display & sensor libs.

---

## Table of contents

1. [Hardware](#hardware)
2. [Wiring](#wiring)
3. [Libraries](#libraries)
4. [Building & Flashing](#building--flashing)
5. [What the device does](#what-the-device-does)
6. [OLED UI](#oled-ui)
7. [Buttons & Buzzer](#buttons--buzzer)
8. [Networking & Backend](#networking--backend)
9. [Captive Portal (Configuration)](#captive-portal-configuration)
10. [Calibration](#calibration)
11. [Storage (NVS keys)](#storage-nvs-keys)
12. [Troubleshooting](#troubleshooting)
13. [Customization](#customization)
14. [Security notes](#security-notes)
15. [License](#license)

---

## Hardware

* **MCU:** ESP32 DevKit (tested with ESP32-WROOM dev boards)
* **Display:** 0.96" or 1.3" SSD1306 OLED, I²C (address `0x3C` default)
* **Temp/Humidity:** DHT11 (GPIO 14)
* **Voltage sense:** Analog input **GPIO 33** via resistor divider
* **Buttons:** GPIO **26** (Manual Override), **27** (Generator Run)
* **Buzzer:** GPIO **25** (active buzzer, low-side or direct)
* **Pull-ups:** Buttons use internal pull-ups

> Note: I²C pins in this sketch are **SDA=GPIO 4, SCL=GPIO 5** to match your code. ESP32 defaults are typically SDA=21, SCL=22; change in `Wire.begin()` if needed.

---

## Wiring

```
ESP32                Peripheral
-----                ----------
3V3                  SSD1306 VCC
GND                  SSD1306 GND
GPIO 4  (SDA)  ----  SSD1306 SDA
GPIO 5  (SCL)  ----  SSD1306 SCL

GPIO 14 (DHT)  ----  DHT11 DATA  (use 10k pull-up to 3V3)
3V3                  DHT11 VCC
GND                  DHT11 GND

GPIO 33 (ADC)  ----  Battery divider output
3V3/GND          ->  Resistor divider (pick based on expected max V)
                     Example: 100k:20k → factor ~ 6.0 (configure via portal)

GPIO 26          ->  Button 1 to GND (Manual Override), INPUT_PULLUP
GPIO 27          ->  Button 2 to GND (Generator Running), INPUT_PULLUP

GPIO 25          ->  Buzzer (+) (or gate), GND to buzzer (–)
```

---

## Libraries

Install these from Arduino Library Manager (or PlatformIO):

* **Adafruit SSD1306**
* **Adafruit GFX**
* **DHT sensor library**
* **ArduinoJson** (v6+)

ESP32 core provides:

* **WiFi.h**, **WebServer.h**, **DNSServer.h**, **Preferences.h**, **HTTPClient.h**
* **Wire.h** (I²C)

---

## Building & Flashing

1. **Board:** `ESP32 Dev Module`
2. **Upload speed:** 115200 (or faster if stable)
3. **Partition scheme:** Default (no special needs)
4. **Flash** the provided sketch.

> If your OLED doesn’t initialize, double-check address (`0x3C` vs `0x3D`) and I²C pins in `Wire.begin(4, 5)`.

---

## What the device does

* On boot:

  * Shows a splash (“GENNY Monitor v2.0”).
  * Loads configuration from NVS.
  * Allows **long-press on BOOT (GPIO0) for 3 s** to enter the **captive portal**.
  * Attempts Wi-Fi connection with stored creds (non-blocking).
  * If no creds or connection fails quickly, **starts captive portal** automatically.

* While **offline**:

  * Shows **OFFLINE** and **only V/T/H** at the bottom.
  * Wi-Fi icon shows **0 bars**.
  * Every **30 s**: attempts reconnect; shows **“Reconnecting…”** briefly.
  * **Buttons are disabled** (error buzz).

* When **online**:

  * Normal UI appears (override/run pills, V/T/H, Wi-Fi bars, sync indicator).
  * Every **30 s**:

    * GETs backend generator state
    * POSTs sensor readings (T/H and Voltage)
  * **Auto control**:

    * If **manual override** is ON → follow backend `generatorRunning`
    * Else use hysteresis on voltage: `< 12.2V → RUN`, `> 13.4V → STOP` (configurable in code)

---

## OLED UI

Top band:

* **Voltage** (left)
* **Wi-Fi bars** (right-left, 0–4)
* **Sync square** (right):

  * **Empty circle** = idle (WAIT)
  * **Up triangle** = GET
  * **Down triangle** = POST
  * **Solid square** = OFF (no Wi-Fi)
  * Blink = short “activity” pulse

Main section (online):

* **OVR** and **RUN** pills (ON/OFF)

Bottom strip:

* **T** (°C), **H** (%)
* **Countdown** to next sync (s)

Offline view:

* “**OFFLINE**”, “**Reconnecting…**” or retry countdown
* Only **V/T/H** displayed

---

## Buttons & Buzzer

* **Button 1 (GPIO 26):** Toggle **manual override** (online only)

  * Short **ack buzz**
  * Triggers `POST /setGeneratorState`
* **Button 2 (GPIO 27):** Toggle **generatorRunning** if **manual override ON** (online only)

  * Two short buzzes for **ON**, one long for **OFF**
  * Triggers `POST /setGeneratorState`
* **Offline:** both buttons cause **error buzz**; no network action.

---

## Networking & Backend

**Retry/Intervals:**

* Reconnect attempt every **30 s** when offline (non-blocking)
* `GET` and `POST` timers: **30 s**
* All network functions **guarded** by `wifiConnected`

**Endpoints (examples):**

* `GET https://domain/_functions/getGeneratorState?gennyName=<urlencoded name>`

  * Header: `x-auth-token: <GENNY_POST_AUTH_TOKEN>`
  * Response JSON (example):

    ```json
    { "manualOverride": true, "generatorRunning": false }
    ```
* `POST https://domain/_functions/setGeneratorState`

  * Header: `Content-Type: application/json`
  * Header: `x-auth-token: <GENNY_POST_AUTH_TOKEN>`
  * Body:

    ```json
    {
      "gennyName": "OfficeGen",
      "generatorRunning": true,
      "manualOverride": false
    }
    ```
* `POST https://domain/_functions/setSensorReading`

  * Header: `Content-Type: application/json`
  * Header: `x-auth-token: <SENSOR_AUTH_TOKEN>`
  * Temp/Humidity body:

    ```json
    { "deviceName": "Office", "temp": 21.5, "humidity": 47 }
    ```
  * Voltage body:

    ```json
    { "deviceName": "officeBattery", "value": 12.83 }
    ```

**Dynamic `getGennyUrl`:**
The GET URL is **built at runtime** using the **current `gennyName`** from configuration. If you change `gennyName` in the portal, subsequent GETs use the new value.

---

## Captive Portal (Configuration)

**When does it start?**

* On **boot**, hold **BOOT** (GPIO0) for **3 s** → portal
* Automatically if Wi-Fi connect fails (or no SSID configured)

**Access point:**

* SSID: **`Genny Monitor`**
* Password: **`GennySetup!`** (change in code: `AP_PASS`)
* Portal address: `http://192.168.4.1/`
* Uses **HTTP Basic Auth** to access the page:

  * Default **username**: `admin`
  * Default **password**: `changeme`

> Change the admin credentials immediately in the portal.

**Configurable fields:**

* Wi-Fi **SSID** / **Password**
* **gennyName**
* **deviceName**
* **batteryDeviceName**
* **Admin username/password**
* **voltageCalFactor** (float, 0.10–50.00)

**Save behavior:**

* Values are written to **NVS/Preferences** and the device **reboots**.

**Timeout:**

* Portal auto-closes after **5 minutes** of inactivity (configurable via `AP_TIMEOUT_MS`).

---

## Calibration

`voltageCalFactor` converts ADC voltage (at GPIO 33) to **actual battery voltage**:

```
batteryVoltage = adcVoltage * voltageCalFactor
adcVoltage = (rawADC / 4095) * 3.3V
```

**To calibrate:**

1. Measure your battery with a **multimeter** (e.g., 12.60 V).
2. Look at the device’s displayed voltage or serial print (e.g., 12.10 V).
3. New factor = old factor × (multimeter / displayed).
   Example: `5.00 × (12.60 / 12.10) = 5.21`
4. Open the **captive portal** → set **Voltage Calibration Factor** to **5.21** and save.

> The correct factor depends on your resistor divider ratio and any tolerances.

---

## Storage (NVS keys)

Namespace: `genny`

| Key     | Type   | Meaning                    |
| ------- | ------ | -------------------------- |
| `ssid`  | string | Wi-Fi SSID                 |
| `pass`  | string | Wi-Fi password             |
| `genny` | string | Genny name                 |
| `dev`   | string | Device name for T/H        |
| `bdev`  | string | Device name for Voltage    |
| `admU`  | string | Admin username (portal)    |
| `admP`  | string | Admin password (portal)    |
| `vcal`  | float  | Voltage calibration factor |

---

## Troubleshooting

**OLED is blank**

* Check `SSD1306` **address** (0x3C vs 0x3D).
* Verify **I²C pins**. This sketch uses **SDA=4, SCL=5**: `Wire.begin(4, 5)`.
* Confirm **power** (3V3) and GND.

**Shows full Wi-Fi bars while offline**

* Fixed: offline path forces **0 bars** and uses **SYNC\_OFF** indicator.

**DHT shows `nan`**

* Check DHT **pull-up** (10k to 3V3 on DATA).
* DHT needs \~1s between reads (we use 1000 ms).

**Voltage reading incorrect**

* Adjust **`voltageCalFactor`** in the portal.
* Verify resistor **divider** ratio and wiring.

**Can’t reach portal**

* Ensure you’re connected to **`Genny Monitor`** AP.
* Try `http://192.168.4.1/` directly.
* Some phones cache DNS; toggle Wi-Fi off/on.

**Forgot portal admin password**

* Reflash or implement a “factory reset” (e.g., long-press both buttons at boot to clear NVS).

---

## Customization

* **I²C pins:** change `Wire.begin(SDA, SCL)`.
* **Buttons:** update pin numbers and labels.
* **Intervals:** tweak `syncIntervalMs`, `sensorPostEveryMs`.
* **Hysteresis:** edit `startVoltage`, `stopVoltage`.
* **AP SSID & password:** change `AP_SSID`, `AP_PASS`.
* **Portal timeout:** `AP_TIMEOUT_MS`.
* **Brand color:** change `BRAND_COLOR` (used in CSS).
* **Endpoints / tokens:** update `POST_GENNY_URL`, `POST_SENSOR_URL`, `GET_GENNY_BASE`, `GENNY_POST_AUTH_TOKEN`, `SENSOR_AUTH_TOKEN`.

---

## Security notes

* The AP is WPA2-PSK; change `AP_PASS` from the default.
* Portal uses **HTTP Basic Auth**; change the default admin credentials on first use.
* Tokens are stored in firmware; treat the device as **trusted hardware** on a trusted network.
* Consider putting the ESP32 on an **IoT VLAN**.

---

## License

MIT (or your preferred license). Include a `LICENSE` file if you plan to distribute.

---

### Appendix: Pin quick reference

| Signal                   | Pin |
| ------------------------ | --- |
| OLED SDA                 | 4   |
| OLED SCL                 | 5   |
| DHT11 DATA               | 14  |
| Battery ADC              | 33  |
| Button 1 (Override)      | 26  |
| Button 2 (Gen Run)       | 27  |
| Buzzer                   | 25  |
| BOOT (config long-press) | 0   |

---

If you want, I can tailor the README for PlatformIO (env, `platformio.ini`), or add a schematic snippet for your exact resistor values and connector layout.
