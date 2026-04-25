# Changelog – Venus OS Home Assistant Addon

All notable changes are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.0.0] – 2026-04-25

### Added
- Initial release of the Venus OS Home Assistant addon.
- Multi-stage Dockerfile based on `ghcr.io/home-assistant/{arch}-base-debian:bookworm`
  (Debian Bookworm + s6-overlay v3 + bashio).
- s6-overlay service units for:
  - `dbus` — private D-Bus system bus (permissive policy for container use)
  - `mosquitto` — MQTT broker on port 1883
  - `dbus-systemcalc-py` — Venus OS system calculator daemon
  - `dbus-mqtt` — D-Bus ↔ MQTT bridge (VRM topic tree for HA auto-discovery)
- `run.sh` initialisation script that reads HA addon options and writes runtime
  configuration files before services start.
- Support for optional MQTT authentication (username/password).
- Auto-generation of a stable VRM Portal ID when none is provided.
- VE.Direct serial port configuration (`vedirect_port`, `vedirect_baud`).
- VE.Can interface configuration with automatic `ip link set … up type can bitrate 250000`.
- VRM Portal API token pass-through to `dbus-mqtt`.
- Optional Modbus TCP server flag (`modbus_enabled`).
- AppArmor security profile covering serial devices, CAN interfaces, and
  required Linux capabilities (`NET_ADMIN`, `NET_RAW`).
- GitHub Actions CI/CD workflow:
  - Validates `config.yaml` structure.
  - Builds multi-arch Docker images (amd64, aarch64, armhf, armv7) via QEMU + buildx.
  - Pushes to GitHub Container Registry on version tags.
  - Smoke-tests: container starts, MQTT port reachable.
- User documentation (`DOCS.md`) covering setup, hardware wiring, and known
  limitations.
