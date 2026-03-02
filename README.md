# IKIA Uppåtvind ESP8266

Custom ESPHome firmware for the **IKEA Uppåtvind** (and Förnuftig) air purifier using the onboard **ESP-12F / ESP8266** microcontroller.  
Auto-discovers in **Home Assistant via MQTT or API**. Control fan speed from HA or the physical button on the unit.  
Speed frequencies are adjustable live from HA without ever reflashing.

> Based on the original [uPurifier](https://github.com/horvathgergo/uPurifier) project by horvathgergo.  
> Fully rewritten for ESPHome and updated for Home Assistant 2024 +.

---

## How It Works

The purifier fan is a **4-pin PWM fan** (24 VDC, 0.5 A):

| Fan pin | Function |
|---------|----------|
| POWER | +24 VDC |
| GND | Ground |
| PWM | Speed control input — the ESP drives this pin |
| FG | Tachometer output — pulses back to the ESP for RPM reading |

The ESP controls fan speed by generating a **square wave** on the PWM pin:
- Duty cycle is fixed at **50 %** at all times.
- Speed is changed by varying the **frequency**: higher Hz = faster fan.
- Three speed levels: **Low (152 Hz) · Medium (225 Hz) · High (300 Hz)** — all tunable from HA.

ESPHome runs on the ESP8266, connects to your WiFi and MQTT broker, and publishes the device to Home Assistant automatically through **MQTT discovery**. You can also adopt it via the **ESPHome API** if you have the ESPHome integration in HA instead of MQTT.

---

## Hardware

| Item | Details |
|------|---------|
| IKEA purifier | Uppåtvind *or* Förnuftig |
| MCU | Custom PCB with ESP-12F (pin-compatible with ESP-12E) |
| Fan | 4-pin PWM · 24 VDC · 0.5 A |
| First-time flash tool | USB-to-TTL serial adapter (see section below) |

---

## PCB Pin Map

| GPIO | PCB label | Function |
|------|-----------|----------|
| GPIO4 | FG | Fan tachometer — pulse counter input, calculates RPM |
| GPIO5 | PWM / CLK | Fan speed control — 50 % duty, variable frequency 1–300 Hz |
| GPIO12 | D3 | **Error LED** — off = system healthy · blinks = WiFi or MQTT fault |
| GPIO13 | SW | **User button** — cycles speed: Off → Low → Medium → High → Off |
| GPIO14 | D2 | **Power LED** — on when fan is running, off when fan is stopped |

---

## LED Indicators

| LED | GPIO | Meaning |
|-----|------|---------|
| D2 | GPIO14 | **Solid on** — fan is running at any speed |
| D2 | GPIO14 | **Off** — fan is stopped |
| D3 | GPIO12 | **Off** — everything is working correctly |
| D3 | GPIO12 | **Blinking** — WiFi disconnected or MQTT broker unreachable |

---

## What You Need

### Software

- [ESPHome](https://esphome.io/guides/installing_esphome) — either the CLI or the Home Assistant add-on  
- **Home Assistant** with the **Mosquitto broker** add-on installed and running  
  *(or any standalone MQTT broker on your network)*

### First-Time Flash Hardware

The ESP-12F on the PCB has no USB port, so the very first firmware upload requires a **USB-to-TTL serial adapter**.  
Common adapters: CP2102 · CH340G · FT232RL

> ⚠️ **Use the 5 V output from the adapter to power the PCB.**  
> The PCB has an onboard 3.3 V voltage regulator — it accepts 5 V on its VCC pin and  
> steps it down internally for the ESP8266.  
> The TX / RX signal lines operate at 3.3 V logic, which all standard adapters support.

**Wiring:**

| USB-TTL adapter pin | PCB header pin |
|---------------------|---------------|
| **5V** | VCC |
| GND | GND |
| TX | RX |
| RX | TX |

Do **not** connect anything else during flashing. Disconnect the 24 V fan supply before wiring the TTL adapter.

---

## First-Time Flash — Step by Step

### Step 1 — Install ESPHome

**Option A — Home Assistant add-on (recommended):**
1. In HA go to **Settings → Add-ons → Add-on Store**
2. Search for **ESPHome** and install it
3. Start it and open the ESPHome dashboard

**Option B — CLI on your PC:**
```bash
pip install esphome
```

---

### Step 2 — Prepare your `secrets.yaml`

Copy `secrets.yaml.template` to `secrets.yaml` in the same folder and fill in your real values:

```yaml
wifi_ssid:     "Your WiFi Network Name"
wifi_password: "your_wifi_password"

# IP address of your Home Assistant / Mosquitto broker
mqtt_broker:   "192.168.1.100"
mqtt_username: "mqtt_user"
mqtt_password: "mqtt_password"

# Protect OTA updates with a strong password
ota_password:  "change_me_to_something_strong"
```

To find your HA IP address: **Settings → System → Network → IP address**

To create an MQTT user: **Settings → Add-ons → Mosquitto broker → Configuration → Logins**

> `secrets.yaml` is listed in `.gitignore` — your credentials will never be accidentally committed.

---

### Step 3 — Put the ESP into Flash Mode

The ESP8266 has two modes: **normal boot** and **flash mode**.  
You must force it into flash mode before uploading firmware for the first time.

The PCB has two buttons: **BOOT** (GPIO0) and **RESET**.

**Procedure — follow this exact order:**

1. **Connect** the USB-to-TTL adapter to the PCB (wiring table above) and plug the adapter into your PC.
2. **Hold** the **BOOT** button down — keep holding it throughout the next steps.
3. While still holding BOOT, **press** the **RESET** button briefly, then **release RESET**.
4. **Release** the **BOOT** button.

The ESP is now in flash mode. The D3 LED will stay off (no firmware running).  
If you see any LED activity after releasing BOOT, repeat the procedure — the sequence must be exact.

---

### Step 4 — Compile and Upload

Pick the YAML file for your model:

```bash
# Uppåtvind
esphome run uppatvind.yaml

# Förnuftig
esphome run fornuftig.yaml
```

ESPHome will:
1. Compile the firmware (~2–3 minutes the first time)
2. Detect the serial port automatically
3. Upload the firmware over serial
4. Reboot the device into normal mode

If ESPHome cannot find the serial port automatically, specify it:
```bash
esphome run uppatvind.yaml --device COM3          # Windows
esphome run uppatvind.yaml --device /dev/ttyUSB0  # Linux / Mac
```

To only compile and get the `.bin` file without flashing:
```bash
esphome compile uppatvind.yaml
# Binary output:
# .esphome/build/upurifier-uppatvind/.pioenvs/upurifier-uppatvind/firmware.bin
```

---

### Step 5 — First Boot

After flashing:
1. Disconnect the USB-to-TTL adapter.
2. Power the PCB normally (via its original purifier power supply).
3. The device will boot, connect to WiFi, then connect to MQTT.
4. **D3 blinks** briefly during boot (WiFi + MQTT connecting) — this is normal.
5. Once connected, **D3 turns off** = healthy. **D2** reflects fan state.

---

### Step 6 — Adopt in Home Assistant

**Via MQTT (recommended):**
1. Go to **Settings → Devices & Services → MQTT**
2. The purifier appears automatically under discovered devices — click **Configure**.

**Via ESPHome API:**
1. Go to **Settings → Devices & Services → Add Integration**
2. Search for **ESPHome**
3. Enter the device IP address and API key (shown in ESPHome logs on first boot)

All future firmware updates can be done **wirelessly via OTA** — no USB-to-TTL adapter needed again.

---

## Home Assistant Entities

| Entity | Type | Notes |
|--------|------|-------|
| Fan | Fan control | On/Off + 3 speeds: Low · Medium · High |
| Low Speed Frequency | Number (config) | PWM Hz for speed 1 — default **152 Hz** |
| Medium Speed Frequency | Number (config) | PWM Hz for speed 2 — default **225 Hz** |
| High Speed Frequency | Number (config) | PWM Hz for speed 3 — default **300 Hz** |
| Fan Speed | Sensor (diagnostic) | Tachometer reading in RPM |
| WiFi Signal | Sensor (diagnostic) | Signal strength in dBm |
| Uptime | Sensor (diagnostic) | Time since last reboot |
| IP Address | Text sensor (diagnostic) | Current IP address |

**Speed frequency tuning:**  
The default Hz values work for most units. If your fan spins too slow or too fast at a given speed, adjust the Hz numbers in HA — changes take effect immediately and are saved to flash. No reflash required.

---

## Physical Button

One button, one press = advance to next speed:

```
Off  →  Low  →  Medium  →  High  →  Off  →  …
```

The button has a 50 ms debounce filter to prevent false triggers.

---

## Fallback — Captive Portal

If the device cannot connect to WiFi (wrong password, network changed) it automatically starts a fallback access point:

| | |
|--|--|
| **SSID** | `uPurifier-Setup` |
| **Password** | `micropythoN` |

Connect to it from any phone or laptop — a configuration page appears automatically.  
Enter your new WiFi credentials and the device reconnects. No reflash needed.  
If the portal does not open automatically, navigate to **http://192.168.4.1** in a browser.

---

## OTA Updates (after first flash)

All future firmware updates are wireless:

```bash
esphome run uppatvind.yaml
```

ESPHome detects the device on the network by hostname and uploads over WiFi. You will be prompted for the OTA password set in `secrets.yaml`.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| D3 blinking, device is online | MQTT broker unreachable | Verify `mqtt_broker` IP and credentials in `secrets.yaml` |
| D3 blinking, no IP assigned | WiFi not connecting | Check SSID/password; use captive portal to update WiFi credentials |
| Fan does not spin | PWM frequency too low for your motor | Increase Low Speed Frequency in HA (try 200–300 Hz) |
| Fan runs at wrong speeds | Default Hz values not matched to motor | Tune the three frequency entities in HA |
| Device not in HA | MQTT discovery not firing | Confirm `discovery: true` in YAML; check Mosquitto is running |
| Serial upload fails — port not found | Wrong COM port | Specify port manually with `--device COMx` |
| Serial upload — timeout / no response | Not in flash mode | Redo the BOOT+RESET procedure exactly as described |
| Serial upload — boot loop / crash | 5 V not on VCC | Check wiring; use the **5V** pin on the adapter, not 3V3 |
| OTA upload fails | Wrong OTA password | Re-flash via serial with corrected `secrets.yaml` |

---

## File Structure

```
uPurifier-ESPHome/
├── uppatvind.yaml          # ESPHome config for Uppåtvind  ← flash this
├── fornuftig.yaml          # ESPHome config for Förnuftig  ← flash this
├── secrets.yaml.template   # Copy to secrets.yaml and fill in credentials
├── .gitignore              # secrets.yaml excluded from git
└── README.md
```

---

## Credits

- Original MicroPython project: [horvathgergo/uPurifier](https://github.com/horvathgergo/uPurifier)
- ESPHome framework: [esphome.io](https://esphome.io)
- Home Assistant: [home-assistant.io](https://www.home-assistant.io)
