# Venus OS – Home Assistant Addon

Run [Victron Energy Venus OS](https://github.com/victronenergy/venus) services
inside a Docker container managed by the Home Assistant Supervisor.

## What this addon does

| Service | Role |
|---|---|
| **dbus-daemon** | Private D-Bus system bus (replaces the host bus) |
| **mosquitto** | MQTT broker — clients connect on port **1883** |
| **dbus-systemcalc-py** | Aggregates values from device daemons (SOC, power, …) |
| **dbus-mqtt** | Bridges D-Bus → MQTT (HA auto-discovery topic tree) |

Home Assistant's built-in **MQTT integration** picks up all Victron entities
automatically using the topic prefix `N/<portalid>/#`, matching the VRM Portal
convention.

---

## Configuration options

### `vrm_id`  _(optional string)_
Your VRM Portal ID (the 12-character hex value printed on your GX device).
Leave blank to let the addon generate and persist one automatically.

### `mqtt_username` / `mqtt_password`  _(optional)_
Credentials for the built-in Mosquitto broker.  Leave both blank to allow
anonymous connections (acceptable on a private HA network; **not** recommended
if the broker port is exposed externally).

### `vedirect_port`  _(optional string)_
Serial device path for a VE.Direct device, e.g. `/dev/ttyUSB0`.
You must also enable **UART** access in the addon configuration.

### `vedirect_baud`  _(optional integer, default `19200`)_
Serial baud rate for the VE.Direct port.

### `vecan_interface`  _(optional string)_
Linux CAN interface name, e.g. `can0`.  The kernel CAN module (e.g. `gs_usb`,
`mcp251x`) **must** already be loaded on the Home Assistant OS host — the addon
cannot install kernel modules.

### `vrm_token`  _(optional password)_
API token for [VRM Portal](https://vrm.victronenergy.com) cloud connectivity.
Obtain one from _VRM → Access Tokens_.

### `modbus_enabled`  _(boolean, default `false`)_
Start the Modbus TCP server on port **502**.  Useful for HA's Modbus
integration or third-party energy-management software.

### `log_level`  _(debug | info | warning | error, default `info`)_
Verbosity of addon log output visible in the HA Supervisor log viewer.

---

## Home Assistant MQTT integration

1. Install and configure the **MQTT** integration (Settings → Devices &
   Services → Add Integration → MQTT).
2. Point it at `localhost:1883` (or the HA host IP if using an external broker).
3. Supply the same credentials you set in the addon options.
4. Venus entities appear automatically under _Devices_ as the addon publishes
   `N/<portalid>/#` topics.

---

## Hardware wiring

### VE.Direct (USB adapter)
Plug a [VE.Direct to USB interface](https://www.victronenergy.com/accessories/ve-direct-to-usb-interface)
into the HA host.  The device typically appears as `/dev/ttyUSB0`.

Set `vedirect_port: /dev/ttyUSB0` and enable **UART** in the addon's device
tab.

### VE.Can (CAN bus hat / USB adapter)
Load the appropriate kernel module on the host, verify that `ip link show can0`
works, then set `vecan_interface: can0`.

> **Note:** The addon uses `ip link set can0 up type can bitrate 250000`
> (the Venus OS default).  If your installation requires a different bitrate,
> open a GitHub issue.

### VE.Bus / MultiPlus / Quattro
VE.Bus communication requires the `mk2dbus` daemon (proprietary, not included).
Connect a [MK3-USB interface](https://www.victronenergy.com/accessories/mk3-usb) and
follow the [Venus OS Large](https://github.com/victronenergy/venus/wiki/venus-os-large)
instructions for installing the binary separately.

---

## Known limitations

* The **Qt/QML Venus GUI** cannot run inside Docker.  The lightweight
  [venus-html5-app](https://github.com/victronenergy/venus-html5-app) web UI
  is served via the Ingress port instead (if installed).
* **Kernel modules** (CAN, USB serial) must be present on the HA OS host.
* The **VRM Portal uplink daemon** (`vrmlogger`) is not included; VRM
  connectivity is provided through `dbus-mqtt` when a `vrm_token` is set.
* VE.Direct device drivers (`dbus-vedirect`) are not publicly available from
  Victron; device data appears on D-Bus only when a compatible driver is added.

---

## Support

* [Venus OS GitHub](https://github.com/victronenergy/venus)
* [Victron Community](https://community.victronenergy.com)
* [This addon source](https://github.com/ZaviiNet/venus)
