# uPurifier — ESPHome Edition (2026)

ESPHome firmware for **IKEA Förnuftig** and **IKEA Uppåtvind** air purifiers fitted with a custom **ESP-12F** PCB.  
Auto-discovers in **Home Assistant** via MQTT. Control fan speed from HA or the physical button. Speed frequencies are tunable live from HA without reflashing.

Based on the original [uPurifier](https://github.com/horvathgergo/uPurifier) project by horvathgergo, fully rewritten for ESPHome and updated for HA 2024+.

---

## Hardware

| Component | Details |
|-----------|---------|
| IKEA purifier | Förnuftig *or* Uppåtvind |
| MCU board | Custom PCB with ESP-12F (ESP-12E compatible) |
| Fan | 4-pin PWM, 24 VDC 0.5 A — POWER · GND · PWM · FG |
| Flash tool (first time only) | USB-to-TTL adapter (3.3 V logic: CP2102, CH340, FTDI) |

---

## Pin Map (verified on PCB)

| GPIO | Label | Function |
|------|-------|----------|
| GPIO4 | FG | Fan tachometer input (pulse counter → RPM) |
| GPIO5 | PWM / CLK | Fan speed control — 50 % duty, variable 1–300 Hz |
| GPIO12 | D3 | Error LED — **off** = healthy, **blinks** = WiFi/MQTT fault |
| GPIO13 | SW | User button — cycles Off → Low → Medium → High → Off |
| GPIO14 | D2 | Power LED — on while fan is running, off when fan is off |

---

## What You Need

### Software
- [ESPHome](https://esphome.io/guides/installing_esphome) (CLI or HA add-on)
- Home Assistant with **Mosquitto broker** add-on (or any MQTT broker)

### First-time flash hardware
The ESP-12F module on the PCB has no built-in USB. You need a **USB-to-TTL adapter** (3.3 V — **do not use 5 V**) wired as follows:

| USB-TTL pin | ESP-12F PCB pin |
|-------------|-----------------|
| 3V3 | VCC |
| GND | GND |
| TX | RX |
| RX | TX |

To enter **flash mode** before powering on:
1. Hold **GPIO0** (FLASH button on PCB) LOW
2. Apply power (or press RESET)
3. Release GPIO0

After the first OTA-capable firmware is flashed, all future updates can be done **wirelessly** via ESPHome OTA.

---

## Setup

### 1. Clone / download this repo

```bash
git clone https://github.com/negoti8za/uPurifier-ESPHome.git
cd uPurifier-ESPHome/esphome
```

### 2. Fill in `secrets.yaml`

Copy `secrets.yaml` and fill in your real credentials:

```yaml
wifi_ssid:     "Your WiFi Network Name"
wifi_password: "your_wifi_password"

mqtt_broker:   "192.168.1.100"   # IP of your HA host / Mosquitto broker
mqtt_username: "mqtt_user"
mqtt_password: "mqtt_password"

ota_password:  "change_me_to_something_strong"
```

> **Never commit `secrets.yaml` with real values.** It is in `.gitignore`.

### 3. Configure Home Assistant MQTT

In HA:  
**Settings → Add-ons → Mosquitto broker → Configuration**

Create an MQTT user that matches `mqtt_username` / `mqtt_password` in your `secrets.yaml`.

### 4. Compile the firmware

Pick the YAML for your model:

```bash
# Förnuftig
esphome compile fornuftig.yaml

# Uppåtvind
esphome compile uppatvind.yaml
```

The compiled binary will be at:
```
.esphome/build/upurifier-fornuftig/.pioenvs/upurifier-fornuftig/firmware.bin
# or
.esphome/build/upurifier-uppatvind/.pioenvs/upurifier-uppatvind/firmware.bin
```

### 5. First-time flash (USB-to-TTL required)

With your USB-to-TTL adapter connected and the device in flash mode:

```bash
esphome run fornuftig.yaml
# or
esphome run uppatvind.yaml
```

ESPHome will compile and upload via the serial port automatically. Once complete, remove the USB-to-TTL adapter — all future updates are wireless.

### 6. Adopt in Home Assistant

After boot the device will:
1. Connect to your WiFi
2. Connect to MQTT and publish its discovery payload
3. Appear automatically in HA under **Settings → Devices & Services → MQTT**

---

## Home Assistant Entities

| Entity | Type | Description |
|--------|------|-------------|
| Fan | Fan | On/off + 3 speeds (Low / Medium / High) |
| Low Speed Frequency | Number (config) | PWM Hz for speed 1 (default 152 Hz) |
| Medium Speed Frequency | Number (config) | PWM Hz for speed 2 (default 225 Hz) |
| High Speed Frequency | Number (config) | PWM Hz for speed 3 (default 300 Hz) |
| Fan Speed | Sensor (diagnostic) | Tachometer RPM |
| WiFi Signal | Sensor (diagnostic) | dBm |
| Uptime | Sensor (diagnostic) | Seconds |
| IP Address | Text sensor (diagnostic) | Current IP |

Speed frequencies are stored in flash and survive reboots. Adjust them live from HA to tune the motor for your unit without reflashing.

---

## Button Behaviour

Single press cycles through:

```
Off → Low → Medium → High → Off → …
```

---

## LED Behaviour

| LED | GPIO | State |
|-----|------|-------|
| D2 (Power) | GPIO14 | **On** when fan is running, **off** when fan is off |
| D3 (Error) | GPIO12 | **Off** when healthy, **blinks** on WiFi or MQTT fault |

---

## Fallback Access Point

If WiFi fails (wrong password, router down) the device will start a fallback AP:

- **SSID:** `uPurifier-Setup`  
- **Password:** `micropythoN`

Connect to it and navigate to `http://192.168.4.1` to update WiFi credentials via the captive portal — no reflash needed.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| D3 blinking, device online | MQTT broker unreachable | Check `mqtt_broker` IP and credentials in `secrets.yaml` |
| Fan not spinning | Wrong PWM frequency for your motor | Adjust the speed frequency number entities in HA |
| Device not appearing in HA | MQTT discovery not enabled | Ensure `discovery: true` in MQTT config; check broker credentials |
| Can't flash via serial | Not in flash mode | Hold GPIO0, cycle power, then release GPIO0 |
| OTA fails | Wrong OTA password | Re-flash via serial with corrected `secrets.yaml` |

---

## Credits

- Original project: [horvathgergo/uPurifier](https://github.com/horvathgergo/uPurifier)
- ESPHome: [esphome.io](https://esphome.io)
