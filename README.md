# Nue Zigbee Light & Fan Controller – Hubitat driver

**Namespace:** `mr_fy_hubitat`
**Model supported:** `LXN-4S27LX1.0` (3A Smart Home / Nue)

Hubitat Elevation driver for the **Nue Zigbee fan + light wall switch**.

## Why this driver

The switch has four Zigbee on/off endpoints: one for the light and one each for the fan's High, Medium and Low relays. Paired with a generic driver, it shows up as **four separate switches**. Nothing ties the three fan switches together as one fan, and none of them gives you a fan speed.

This driver presents the hardware the way you use it: **one light and one fan with a speed**.

| Endpoint | Function |
|---|---|
| 01 | Light |
| 02 | Fan – High |
| 03 | Fan – Medium |
| 04 | Fan – Low |

The fan relays are interlocked in the switch, so only one speed is on at a time. The driver reads all three endpoints and publishes a single speed: `off`, `low`, `medium` or `high`. When the switch reports several fan endpoints in a burst, it waits briefly for them to settle before publishing, so the speed doesn't flicker.

## Features

- **Two child devices**, created with Hubitat's built-in component drivers:
  - **`<device name>-Light`** – a Generic Component Switch.
  - **`<device name>-Fan`** – a Generic Component Fan Control with Off, Low, Medium and High. Turning it "on" without a speed sets **Medium**.
- **Parent device** – also exposes Light and Fan Control. Its on/off controls the light, and it has a `setSpeed` command for the fan.
- **Speed cycling** – `cycleSpeed` steps Off → Low → Medium → High → Off (on the Fan child and the parent).
- **Stays in sync with the wall buttons** – on/off reporting is configured on all four endpoints (at most every 10 minutes when nothing changes), so physical presses update Hubitat.
- **`lastCheckin`** – time of the last message from the switch.

## Installation

1. In Hubitat, go to **Drivers Code → New Driver → Import**, paste this raw URL, then **Save**:
```
https://raw.githubusercontent.com/mrfy8/hubitat-nue-zigbee-light-fan-controller/main/nue-zigbee-fan-light.groovy
```
2. Pair the switch (Devices → Add Device → Zigbee, then put the switch in pairing mode by holding a button for 5–10 seconds).
3. Hubitat should pick this driver automatically from its fingerprint. If not, open the device and set **Type** to `Nue Zigbee Light Fan Controller`.
4. On the device page, click **Configure**. This creates the Light and Fan child devices and sets up reporting.

If the child devices don't appear, click **Configure** again. **After updating the driver, click Configure once.**

## Settings

| Setting | Default | Notes |
|---|---|---|
| Enable debug logging | Off (on for 30 minutes after first install) | Turns itself off after 30 minutes |
| Enable descriptionText logging | On | Logs light and fan changes |

## Device fingerprint

| | |
|---|---|
| Model | `LXN-4S27LX1.0` |
| Manufacturer | `3A Smart Home DE` |
| Profile ID | `C05E` |
| Endpoint ID | `01` |
| In clusters | `0000,0004,0003,0006,0005,1000,0008` |

## Disclaimer

Community driver, not affiliated with Nue or 3A Smart Home. Use at your own risk.
