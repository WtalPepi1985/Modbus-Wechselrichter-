# Offgridtec PSI Wechselrichter – ESPHome Modbus Integration

ESPHome-Konfiguration für den **Offgridtec® PSI Sinus-Spannungswandler** (2000W/3200W, 24V → 230V) mit RS485/Modbus-Anbindung über einen ESP32-C3 und vollständiger Home Assistant Integration.

![ESPHome](https://img.shields.io/badge/ESPHome-ESP32--C3-blue?logo=esphome)
![HA](https://img.shields.io/badge/Home%20Assistant-Integration-41bdf5?logo=homeassistant)
![Modbus](https://img.shields.io/badge/Protocol-Modbus%20RTU-orange)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Hardware

| Komponente | Details |
|---|---|
| **Wechselrichter** | Offgridtec PSI 2000W/3200W 24V, MPN 011255 |
| **Mikrocontroller** | ESP32-C3 DevKitM-1 |
| **RS485-Modul** | MAX3485 oder ähnlich (3.3V-kompatibel) |
| **Display** | SSD1306 OLED 128×64 (I2C) |
| **Status-LEDs** | WS2812B Streifen, 8 LEDs |
| **Taster** | Momentary-Button (Einschalten/Ausschalten) |

---

## Features

- **Echtzeit-Monitoring** via Modbus RTU (Modbus-Adresse 3, 115200 Baud)
  - DC-Eingangsspannung, -Strom und -Leistung
  - AC-Ausgangsspannung, -Strom und -Leistung
  - Innentemperatur + MOSFET-Temperatur
  - Errechneter Wirkungsgrad in Prozent
- **Steuerung** über Home Assistant
  - Remote Mode aktivieren/deaktivieren
  - Wechselrichter ein-/ausschalten
- **OLED-Display** (SSD1306 128×64) mit Live-Anzeige
  - DC/AC-Spannung, Ausgangsleistung
  - Akkustand und Solar-Leistung aus HA
  - Fehlermeldungen bei Modbus- oder WR-Fehler
  - Boot-Splash „Powered by Pogrzeba"
- **WS2812B Status-LEDs** (8 LEDs)
  - Grün rotierend = WR aktiv
  - Blau = Standby
  - Rot blinkend = kein Modbus / kein HA
  - Rot dauerhaft = WR-Fehler
  - Regenbogen = Booting
- **BLE Proxy** für Home Assistant Bluetooth-Reichweite
- **Physischer Taster** zum Ein-/Ausschalten (Toggle)
- **Watchdog**: Modbus-Timeout-Erkennung (>15s kein Update → Fehlerstate)
- **BLE-Reset** alle 15 Minuten + manuell per HA-Button

---

## Pinbelegung (ESP32-C3)

| GPIO | Funktion |
|---|---|
| GPIO 4 | I2C SDA (OLED) |
| GPIO 5 | I2C SCL (OLED) |
| GPIO 6 | Modbus Flow-Control (DE/RE) |
| GPIO 7 | WS2812B Datenleitung |
| GPIO 10 | Taster (INPUT, mit Debounce 50ms) |
| GPIO 20 | UART RX (RS485 → ESP) |
| GPIO 21 | UART TX (ESP → RS485) |

---

## Modbus-Register

Modbus-Adresse: **3** | Baudrate: **115200** | Parity: **None** | Stop-Bits: **1**

| Register | Adresse | Typ | Skalierung | Einheit | Beschreibung |
|---|---|---|---|---|---|
| DC Spannung | 0x3108 | Read / U_WORD | × 0.01 | V | Eingangsspannung |
| DC Strom | 0x3109 | Read / U_WORD | × 0.01 | A | Eingangsstrom |
| DC Leistung | 0x310A | Read / U_WORD | × 0.01 | W | Eingangsleistung |
| AC Spannung | 0x310C | Read / U_WORD | × 0.01 | V | Ausgangsspannung |
| AC Strom | 0x310D | Read / U_WORD | × 0.01 | A | Ausgangsstrom |
| AC Leistung | 0x310E | Read / U_WORD | × 0.01 | W | Ausgangsleistung |
| Temperatur | 0x3111 | Read / U_WORD | × 0.01 | °C | Innentemperatur |
| MOSFET Temp | 0x3112 | Read / U_WORD | × 0.01 | °C | MOSFET-Temperatur |
| Status | 0x3202 | Read / U_WORD | Bitfeld | – | Statusbits (s.u.) |
| Remote Mode | 0x0011 | Coil | – | – | Remote-Steuerung aktivieren |
| WR Ein/Aus | 0x000F | Coil | – | – | Wechselrichter schalten |

### Statusbits (0x3202)

| Bit | Bedeutung |
|---|---|
| Bit 0 | WR aktiv (Working) |
| Bit 1 | Fehler allgemein |
| Bit 5 | Überlast |
| Bit 7 | Übertemperatur |
| Bit 8 | Überspannung Eingang |
| Bit 10 | Unterspannung Eingang |
| Bit 11 | Kurzschluss Ausgang |

---

## Schaltplan (vereinfacht)

```
Offgridtec PSI (RS485)          MAX3485 Modul              ESP32-C3
┌─────────────────┐            ┌──────────────┐            ┌──────────┐
│  RS485  A (+)  ─┼────────────┼─ A           │            │          │
│  RS485  B (-)  ─┼────────────┼─ B           │            │          │
│  GND           ─┼────────────┼─ GND         │            │          │
└─────────────────┘            │  RO  ────────┼────────────┼─ GPIO20  │
                               │  DI  ────────┼────────────┼─ GPIO21  │
                               │  DE/RE ──────┼────────────┼─ GPIO6   │
                               │  VCC  ───────┼── 3.3V     │          │
                               └──────────────┘            └──────────┘
```

---

## Installation

### Voraussetzungen

- [ESPHome](https://esphome.io) installiert (>= 2024.x)
- Home Assistant mit ESPHome Add-on
- `secrets.yaml` mit WiFi-Zugangsdaten

### secrets.yaml

```yaml
wifi_ssid: "DeinNetzwerk"
wifi_password: "DeinPasswort"
```

### Kompilieren & Flashen (OTA)

```bash
esphome run --device 10.10.5.241 garage-wr-modbus.yaml
```

### Erstflash (USB)

```bash
esphome compile garage-wr-modbus.yaml
esptool --port /dev/ttyACM0 --chip esp32c3 --before default_reset --after hard_reset \
  write_flash 0x0 .esphome/build/garage-wr-modbus/.pioenvs/garage-wr-modbus/firmware.factory.bin
```

---

## Home Assistant Entitäten

Nach der Integration erscheinen automatisch folgende Entitäten:

| Entität | Typ | Beschreibung |
|---|---|---|
| `sensor.garage_wr_modbus_psi_dc_spannung` | Sensor | DC-Eingangsspannung (V) |
| `sensor.garage_wr_modbus_psi_dc_strom` | Sensor | DC-Eingangsstrom (A) |
| `sensor.garage_wr_modbus_psi_dc_leistung` | Sensor | DC-Eingangsleistung (W) |
| `sensor.garage_wr_modbus_psi_ac_spannung` | Sensor | AC-Ausgangsspannung (V) |
| `sensor.garage_wr_modbus_psi_ac_strom` | Sensor | AC-Ausgangsstrom (A) |
| `sensor.garage_wr_modbus_psi_ac_leistung` | Sensor | AC-Ausgangsleistung (W) |
| `sensor.garage_wr_modbus_psi_temperatur` | Sensor | Innentemperatur (°C) |
| `sensor.garage_wr_modbus_psi_mosfet_temperatur` | Sensor | MOSFET-Temperatur (°C) |
| `sensor.garage_wr_modbus_psi_wirkungsgrad` | Sensor | Wirkungsgrad (%) |
| `switch.garage_wr_modbus_psi_wechselrichter` | Switch | WR Ein/Aus |
| `switch.garage_wr_modbus_psi_remote_mode` | Switch | Remote-Steuerung |
| `button.garage_wr_modbus_ble_reset` | Button | BLE-Scan neu starten |
| `light.garage_wr_modbus_psi_status_led` | Light | WS2812B Status-LEDs |

---

## Wechselrichter-Spezifikationen

| Eigenschaft | Wert |
|---|---|
| Eingangsspannung | 21,6 – 32 V DC |
| Ausgangsspannung | 220/230 V AC (einstellbar) |
| Dauerleistung | 1600 W |
| 15-min-Leistung | 2000 W |
| Spitzenleistung | 3200 W |
| Wellenform | Reiner Sinus (16-Bit SPWM) |
| Frequenz | 50/60 Hz (DIP-Schalter) |
| Kommunikation | RS485, Modbus RTU |
| Schutzfunktionen | Überspannung, Unterspannung, Überlast, Kurzschluss, Übertemperatur |
| Abmessungen | 326,5 × 231,5 × 98,5 mm |
| Gewicht | 4,6 kg |

Produktseite: [Offgridtec PSI 2000W/3200W 24V](https://www.offgridtec.com/offgridtecr-psi-sinus-spannungswandler-rs485-2000w-3200w-24v-230v.html)

---

## Lizenz

MIT License – frei verwendbar, Änderungen willkommen.
