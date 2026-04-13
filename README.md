# CTEK CS ONE Gen 2 — ESPHome BLE Gateway

An ESPHome configuration that creates a BLE gateway for the **CTEK CS ONE Gen 2** battery charger, exposing charging data as sensors in Home Assistant.

This is believed to be the first open-source integration for the CTEK CS ONE Gen 2. The device launched in January 2026 and uses BLE 4.2 Secure Connections with MITM-protected bonding, which required reverse engineering the GATT protocol via Android HCI snoop logs and Bluetooth LE Explorer.

---

## Hardware Required

- **CTEK CS ONE Gen 2** battery charger
- **ESP32 board** with BLE support (see notes below)
- USB-C power supply for the ESP32

### ESP32 Board Requirements

Any ESP32 board with BLE support should work. The minimum requirements are:

- ESP32 with BLE 4.2 or later
- Sufficient flash for ESPHome firmware (~2MB minimum, 4MB recommended)
- WiFi support

This configuration was developed and tested on a **Waveshare ESP32-C3-Zero-M**. If you use a different board, note that:

- **ESP32-C3, S2, S3, C6** variants require `framework: esp-idf` and may need other minor adjustments
- **Original ESP32 (Xtensa)** can use `framework: arduino` which may require changes to the BLE lambda syntax and LED platform
- The LED configuration assumes a WS2812 RGB LED — if your board has a different LED or none at all, remove or adjust the `light` section

---

## Features

The following sensors are exposed to Home Assistant:

| Sensor | Unit | Notes |
|--------|------|-------|
| Battery Voltage | V | Sampled voltage at battery terminals |
| Charger Voltage Out | V | Output voltage from charger |
| Charge Current | A | Current being delivered |
| Charger Temperature | °C | Internal transformer temperature |
| Battery SOC | % | State of charge estimate |
| Estimated Battery Capacity | Ah | CTEK's auto-detected capacity |
| Voltage Setpoint | V | Target charge voltage |
| Current Setpoint | A | Target charge current |
| Remaining Time | h | Estimated time to full charge |
| Total Charge Time | s | Cumulative charge time |
| Charge Status | - | Raw status value (decoding in progress) |
| Detected Battery Chemistry | - | Raw chemistry value (decoding in progress) |
| Available Programs | text | e.g. APTO, RECOND, SUPPLY |
| Firmware Version | text | Charger firmware version string |

Controls:
- **CTEK BLE Connection** switch — enable/disable BLE connection
- **Status LED** — WS2812 RGB LED (green = connected, red = disconnected)

---

## How It Works

The ESP32 acts as an authenticated BLE GATT gateway:

1. Scans for the CTEK CS ONE via `esp32_ble_tracker`
2. Connects and requests MITM-encrypted bonding via `esp_ble_set_encryption`
3. Responds to the passkey request automatically using the configured PIN
4. Polls GATT characteristics from service `f00a0001-d91a-4679-9261-542c9bc01626`
5. Exposes all values as Home Assistant sensors via the ESPHome API

The BLE connection is exclusive — the CTEK app on your phone cannot connect while the ESP32 is connected. Close the app to allow the ESP32 to connect, or disable the CTEK BLE Connection switch in HA temporarily.

---

## Installation

### Prerequisites

- [ESPHome](https://esphome.io) installed (CLI or dashboard)
- Home Assistant with ESPHome integration
- Your CTEK CS ONE Gen 2 PIN (shown during initial app pairing)
- Your CTEK's Bluetooth MAC address (visible in the CTEK app or a BLE scanner)

### Setup

1. Clone this repository:
   ```bash
   git clone https://github.com/yourusername/ctek-cs1-esphome
   cd ctek-cs1-esphome
   ```

2. Create your `secrets.yaml` (not included in this repository):
   ```yaml
   wifi_ssid: "YourWiFiSSID"
   wifi_password: "YourWiFiPassword"
   api_encryption_key: "YourAPIEncryptionKeyHere"
   ota_password: "YourOTAPasswordHere"
   fallback_password: "YourFallbackPasswordHere"
   ctek_mac: "XX:XX:XX:XX:XX:XX"
   ctek_passkey: "000000"
   ```

3. Generate an API encryption key if you don't have one:
   ```bash
   esphome generate-secret
   ```

4. For the first flash, put your ESP32 into bootloader mode (typically hold BOOT while connecting USB), then compile:
   ```bash
   esphome compile ctek_cs1_esphome.yaml
   ```
   Flash `firmware.factory.bin` via [https://web.esphome.io](https://web.esphome.io)

5. For subsequent updates, use OTA:
   ```bash
   esphome run ctek_cs1_esphome.yaml
   ```

### First Connection & Bonding

On first boot the ESP32 will attempt to bond with the CTEK. The passkey exchange happens automatically using the PIN in `secrets.yaml`. The bond is stored in the ESP32's flash and subsequent connections don't require re-pairing.

If bonding fails, power cycle the CTEK charger and restart the ESP32.

---

## Adding to Home Assistant

Once flashed and running, the device will be auto-discovered by HA via the ESPHome integration. Accept the device and it will appear under **Settings → Devices & Services → ESPHome**.

All sensors are grouped under a single **CTEK CS ONE** device.

---

## Known Issues & Unknowns

- **Charge Status** reads a raw numeric value — the meaning of individual bits is not yet decoded. Capturing values at different charge states (bulk, absorption, float, supply mode) would help decode this.
- **Charger State** characteristic (`f00a0037`) is not always found during service discovery — under investigation.
- **Detected Battery Chemistry** returns a raw number — mapping to chemistry types (AGM, GEL, lithium etc.) not yet confirmed.
- **Estimated Battery Capacity** may not match actual battery capacity until the CTEK has completed several charge cycles in APTO mode.
- The CTEK app and this gateway cannot be connected simultaneously.

---
<img width="1036" height="1150" alt="image" src="https://github.com/user-attachments/assets/1882f4ab-b16e-487d-83e7-79582145c8e4" />
---

## Reverse Engineering Notes

The integration was developed by:
1. Capturing BLE HCI snoop logs from an Android device using `adb bugreport`
2. Using **Bluetooth LE Explorer** on Windows to enumerate all GATT services, characteristics and UUIDs with human-readable descriptions
3. Cross-referencing characteristic handles from the HCI logs with the UUID map from Bluetooth LE Explorer to identify sensor data
4. Iterating on ESPHome config to handle BLE 4.2 Secure Connections authentication

All data characteristics are big-endian uint16 values unless otherwise noted. The 4-byte Total Charge Time is big-endian uint32.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

## Contributing

PRs welcome, particularly for:
- Decoding the Charge Status bitfield
- Identifying the Charger State characteristic
- Mapping Detected Battery Chemistry values
- Testing with different battery types and charge modes
- Testing on different ESP32 board variants
