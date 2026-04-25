# ── Stage 1: clone Victron open-source services ──────────────────────────────
FROM debian:bookworm-slim AS builder

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        git \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/victron

# Pin each service to a known-good commit for reproducible builds.
# Update these SHAs when upgrading; the ARGs allow CI overrides.
ARG VELIB_PYTHON_REF=master
ARG DBUS_SYSTEMCALC_PY_REF=master
ARG DBUS_MQTT_REF=master
ARG DBUS_RECORDER_REF=master
ARG DBUS_MODBUS_CLIENT_REF=master

RUN git clone --depth=1 --branch "${VELIB_PYTHON_REF}" \
        https://github.com/victronenergy/velib_python.git \
    && git clone --depth=1 --branch "${DBUS_SYSTEMCALC_PY_REF}" \
        https://github.com/victronenergy/dbus-systemcalc-py.git \
    && git clone --depth=1 --branch "${DBUS_MQTT_REF}" \
        https://github.com/victronenergy/dbus-mqtt.git \
    && git clone --depth=1 --branch "${DBUS_RECORDER_REF}" \
        https://github.com/victronenergy/dbus-recorder.git \
    && git clone --depth=1 --branch "${DBUS_MODBUS_CLIENT_REF}" \
        https://github.com/victronenergy/dbus-modbus-client.git \
    || true   # dbus-modbus-client is optional; continue if unavailable

# ── Stage 2: runtime image ────────────────────────────────────────────────────
# BUILD_FROM is injected by build.yaml / the HA builder per target arch.
ARG BUILD_FROM
FROM ${BUILD_FROM}

# ── System packages ───────────────────────────────────────────────────────────
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        dbus \
        mosquitto \
        mosquitto-clients \
        python3 \
        python3-dbus \
        python3-gi \
        python3-setuptools \
        python3-pip \
        gir1.2-glib-2.0 \
        libglib2.0-0 \
        iproute2 \
        netcat-openbsd \
        socat \
        jq \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# ── Python packages not in apt ────────────────────────────────────────────────
# paho-mqtt: required by dbus-mqtt
# pymodbus: required by dbus-modbus-client (Modbus TCP server, optional)
RUN pip3 install --break-system-packages --no-cache-dir \
    "paho-mqtt>=1.6,<2.0" \
    "pymodbus>=3.6,<4.0"

# ── Victron services (copied from builder stage) ──────────────────────────────
COPY --from=builder /opt/victron /opt/victron

# All Victron daemons import from velib_python; export PYTHONPATH globally.
ENV PYTHONPATH=/opt/victron/velib_python

# ── rootfs overlay (D-Bus config, s6 services, helper scripts) ───────────────
COPY rootfs /

# ── Make all service scripts executable ──────────────────────────────────────
RUN chmod a+x \
        /run.sh \
        /etc/s6-overlay/s6-rc.d/init-addon/up \
        /etc/s6-overlay/s6-rc.d/dbus/run \
        /etc/s6-overlay/s6-rc.d/mosquitto/run \
        /etc/s6-overlay/s6-rc.d/dbus-systemcalc-py/run \
        /etc/s6-overlay/s6-rc.d/dbus-mqtt/run \
    && mkdir -p /data/mosquitto/db /data/venus /etc/venus /etc/dbus-mqtt

# ── OCI labels ────────────────────────────────────────────────────────────────
LABEL \
    io.hass.name="Venus OS" \
    io.hass.description="Victron Energy Venus OS services for Home Assistant" \
    io.hass.arch="${BUILD_ARCH}" \
    io.hass.type="addon" \
    io.hass.version="1.0.0" \
    maintainer="ZaviiNet <https://github.com/ZaviiNet>"
