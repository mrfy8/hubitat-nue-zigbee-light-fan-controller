# Nue Zigbee Light & Fan Controller - Hubitat Driver
**Namespace:** mr_fy_hubitat  
**Model Supported:** LXN-4S27LX1.0 (3A Smart Home / Nue)

A dedicated Hubitat driver for the Nue Zigbee Fan + Light switch. This driver manages the complex multi-endpoint nature of the hardware, providing a clean interface for both light and fan speed control.

## Features
- **Endpoint Separation:** Correctly maps Endpoint 01 to the Light and Endpoints 02-04 to Fan Speeds.
- **Component Device Support:** Automatically creates two child devices using Hubitat's native "Generic Component" drivers.
  - **Light Child:** Appears as a standard Switch.
  - **Fan Child:** Appears as a Fan Control (Off, Low, Medium, High).
- **Speed Cycling:** Includes a `cycleSpeed` helper to move through fan states.
- **Zigbee Reporting:** Configures native reporting for state changes, ensuring the Hub stays in sync with physical wall-button presses.

## Installation
1. Log into your Hubitat Elevation hub.
2. Go to **Drivers Code** > **New Driver**.
3. Click **Import** and paste the Raw URL for `nue-zigbee-fan-light.groovy`.
4. Click **Save**.
5. Pair your Nue device (put it in pairing mode by holding a button for 5-10s).
6. Once found, Hubitat should automatically select this driver (thanks to the fingerprint). 
7. Go to the device page and click **Configure** to spawn the child Light and Fan devices.

## Device Fingerprint
- **ProfileID:** C05E
- **EndpointID:** 01
- **Model:** LXN-4S27LX1.0
- **Manufacturer:** 3A Smart Home DE

---
*Note: If the child devices do not appear immediately, click the 'Initialize' and 'Configure' buttons on the parent device page.*
