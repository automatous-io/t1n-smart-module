# T1N Smart Module

Native Apple HomeKey + HomeKit NFC enabled smart lock system for the T1N series Sprinter that reads real lock state from the factory CTM. Optional integration into Home Assistant to mirror door ajar states for additional automation. Local and remote lock capability anywhere your van gets internet - all from your favorite Apple device.

Long live the million mile Sprinter!

![Dash module — exterior windshield view](media/photos/t1n-smart-module-proto-v1-windshield-install.png)

---

## Why

Factory T1N fobs are unreliable with zero lock-state feedback. Aftermarket kits aren't any better. 

This costs a fraction of what a replacement fob does and can be integrated easily into your van, allowing for multiple smart options that far surpass the shortcomings of the original factory system.

---

## Features

- **Apple HomeKey** — tap iPhone or Apple Watch on the NFC reader to lock/unlock, even in Express Mode when Apple device is 'dead'
- **Native HomeKit** via [HAP-ESPHome](https://github.com/rednblkx/HAP-ESPHome)
- **Siri enabled** - ask Siri to toggle the locks using just your voice
- **Home Assistant** compatible for optional door ajar sensor integration into Apple Home as HomeKit accessory
- **True lock/door ajar - state detection** — PC817 optocouplers read CTM LED sense lines (galvanically isolated)
- **Independent Driver Door & Passenger Door control**
- **Reversible install** — Cat6/RJ45, terminal block to master lock switch interconnect, remove in minutes
- **OTA updates** via ESPHome
- **Remote capable** — operates from anywhere the van has internet, but not required for local control
- **Powered via aux battery** - zero chance of draining your engine battery

---

## Architecture

```
DASH MODULE
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ┌──────────────┐       SPI       ┌──────────────┐             │
│   │  XIAO        │◄───────────────►│   PN532      │             │
│   │  ESP32-C6    │                 │   NFC Module │             │
│   └──────┬───────┘                 └──────────────┘             │
│          │                                                      │
└──────────┼──────────────────────────────────────────────────────┘
           │  Cat6 / RJ45
           │   - 2x trigger (transistor-switched)
           │   - 2x LED sense (optocoupler-isolated)
           │   - 5V + GND
┌──────────┼──────────────────────────────────────────────────────┐
│ INTERFACE BOX (behind center console)                           │
│   ┌──────▼───────┐                                              │
│   │  Terminal    │──► W/Y  → Driver Door lock trigger → CTM     │
│   │  Block       │──► W/R  → Passenger Door lock trigger → CTM  │
│   │  Connections │◄── W/B  ← Driver Door LED sense ← CTM        │
│   │ +            │◄── W/DG ← Passenger Door LED sense ← CTM     │
│   │ -            │ ACC1 (unused spare)                          │
│   └──────────────┘ ACC2 (unused spare)                          │
│   + Aux 5V buck converter + 2A fuse                             │
|   - Aux 5V buck converter + chassis GND                         |
└─────────────────────────────────────────────────────────────────┘
```

**Lock control:** 2N2222A NPN transistors pulse CTM switch inputs to ground (500ms pulse, 1200ms cooldown).
**State detection:** PC817 optocouplers on CTM LED driver lines — `INPUT_PULLUP`, inverted.

> Wire colors reference the **2005 Dodge Sprinter**. Verify against your vehicle.

---

## Lock State Truth Loop

Runs every 250ms. Each door evaluated independently.

| LED State | Condition | Result |
|---|---|---|
| SOLID | ON 500ms+ | Locked |
| BLINK | Edge within 1500ms | Ajar / unlocked |
| OFF | Other LED active | Unlocked (CTM awake) |
| OFF | Was ON < 5s ago | Unlocked (transition) |
| OFF | Both dark > 5s | CTM sleep — hold last known state |

---

## In Action

### HomeKey in Wallet

![HomeKey card for Mabel in Apple Wallet](media/photos/t1n-smart-module-homekey-wallet.png)

### Locked

![Apple Home — Driver Door Locked, Passenger Door Locked](media/photos/t1n-smart-module-home-locked.png)
![MLS — both LEDs on, locked](media/photos/t1n-smart-module-mls-locked.png)

### Unlocked

![Apple Home — Driver Door Unlocked, Passenger Door Unlocked](media/photos/t1n-smart-module-home-unlocked.png)
![MLS — no LEDs, unlocked](media/photos/t1n-smart-module-mls-unlocked.png)

### Passenger Door Open (Door ajar via Home Assistant as binary sensor, Passenger LED blinking)

![Apple Home — Driver Door Closed, Passenger Door Open](media/photos/t1n-smart-module-home-unlocked-pass-open.png)
![MLS — bottom LED on, passenger door ajar](media/photos/t1n-smart-module-mls-unlocked-pass-open.png)

---

## Hardware

| Component | Qty | Purpose |
|-----------|:---:|---------|
| [XIAO ESP32-C6](https://wiki.seeedstudio.com/xiao_esp32c6_getting_started/) | 1 | MCU |
| PN532 NFC (SPI) | 1 | HomeKey |
| PC817 Optocoupler | 2 | LED sense isolation |
| 2N2222A NPN | 2 | Lock pulse output |
| 1k Resistor | 2 | Current limiting, octocoupler  |
| 2.2k Resistor | 2 | Current limiting, transistor |
| Cat6 RJ45 Jack | 2 | Interconnect |
| 12V→5V Buck | 1 | Power |
| 2A Fuse | 1 | Protection |
| Terminal Block | 1 | CTM connections |

### GPIO

| Pin | Function |
|-----|----------|
| GPIO1 | Driver Door pulse (2N2222A base) |
| GPIO2 | Passenger Door pulse (2N2222A base) |
| GPIO18 | SPI MOSI (PN532) |
| GPIO19 | SPI CLK (PN532) |
| GPIO20 | SPI MISO (PN532) |
| GPIO21 | SPI CS (PN532) |
| GPIO22 | Driver Door LED sense (PC817) |
| GPIO23 | Passenger Door LED sense (PC817) |

![XIAO ESP32-C6 pinout reference](media/photos/esp32-c6-pinout-diagram.png)

### CTM Wiring (2005 Dodge Sprinter)

| Wire | Function | Direction |
|---|---|---|
| White/Yellow | Driver Door lock trigger (all doors) | Module → CTM |
| White/Red | Passenger Door lock trigger | Module → CTM |
| White/Black | Driver Door LED sense | CTM → Module |
| White/Dark Green | Passenger Door LED sense | CTM → Module |
| Brown | Ground | Common |

![Factory MLS wiring diagram — 8W-61 Power Door Locks](media/photos/mb-shop-manual-mls-wiring-diagram.png)

### Locks

| Lock | HomeKit | HomeKey (NFC) |
|------|:-------:|:-------------:|
| **Driver Door** | Yes | Yes |
| **Passenger Door** | Yes | No |

Door ajar sensors are exposed to Home Assistant as `binary_sensor` with `device_class: door` and are detected as a door being open or closed. Passenger, slider, and cargo doors are all connected and considered passenger.

---

## Prototype V1

### Initial Proof of Concept

![Intial POC - Breadboard](media/photos/t1n-smart-module-initial-breadboard.png)

### Enclosure

Two 3D-printed enclosures: an angled dash module with NFC face and magnetic lid, and a compact interface box with RJ45 jack and labeled terminal block.

![Enclosure — back / ports view](media/photos/t1n-smart-module-enclosure-proto-v1-1.png)

![Enclosure — NFC face and interface box terminal labels](media/photos/t1n-smart-module-enclosure-proto-v1-2.png)

![Enclosure — angle showing USB-C and terminal block](media/photos/t1n-smart-module-enclosure-proto-v1-3.png)

![Enclosure — magnetic lid removed, embedded magnets](media/photos/t1n-smart-module-enclosure-proto-v1-4.png)

### Modular Perfboard

![PN532 NFC module with ribbon cable to ESP32-C6](media/photos/t1n-smart-module-board-proto-v1-1.png)

![Interface boards — top and bottom wiring](media/photos/t1n-smart-module-board-proto-v1-2.png)

![Assembled stack — ESP32-C6 mounted on interface board](media/photos/t1n-smart-module-board-proto-v1-3.png)

### Van Installation

![Interface box installed behind center console](media/photos/t1n-smart-module-proto-v1-console-install.png)

![Dash module — interior mount](media/photos/t1n-smart-module-proto-v1-dash-install.png)

![Clean install — full cab view](media/photos/t1n-smart-module-proto-v1-clean-install.png)

---

## Flashing

```bash
gh repo clone automatous-io/t1n-smart-module
cd firmware
cp secrets.yaml.example secrets.yaml   # fill in your credentials
esphome run t1n_smart_module.yaml
```

Pair in Apple Home with setup code `159-35-728`.

---

## Dependencies

- [ESPHome](https://esphome.io/)
- [HAP-ESPHome](https://github.com/rednblkx/HAP-ESPHome) — HomeKit + HomeKey
- [Apple Home Hub](https://support.apple.com/en-us/102557)
- [Home Assistant](https://www.home-assistant.io/) (optional)

---

## Disclaimer

Interfaces with vehicle lock systems. Use at your own risk. Verify wiring on your vehicle. Not affiliated with Daimler, Mercedes-Benz, Dodge, Stellantis, or Apple. Made with duct tape and bird shit.
