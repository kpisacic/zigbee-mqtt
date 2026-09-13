# zigbee-mqtt

Lightweight Zigbee backend for a Raspberry Pi 3B+ (1GB RAM), replacing Home
Assistant + deCONZ with **Zigbee2MQTT** + **Mosquitto**. Home Assistant's newer
versions no longer run well on 1GB of RAM; this stack is dramatically lighter
since it has no recorder/history database or integration framework — just
Zigbee devices publishing state over MQTT.

Phase one scope: read-only display of sensor/switch values. No command/control
back to devices yet.

## Stack

- **Mosquitto** — MQTT broker (`eclipse-mosquitto:2`)
- **Zigbee2MQTT** — Zigbee coordinator software (`koenkk/zigbee2mqtt`), talking
  to a ConBee II USB stick over its `deconz` adapter driver, with its built-in
  web frontend for live device values
- Coordinator network is freshly formed (not migrated from deCONZ) — see
  [Notes](#notes) below.

Estimated combined RAM footprint on the Pi: roughly 250–400MB (Mosquitto
~5-10MB, Zigbee2MQTT ~100-180MB, Docker engine overhead ~50-100MB, OS idle
~80-120MB), leaving comfortable headroom on a 1GB board.

## Layout

```
docker-compose.yml
mosquitto/config/mosquitto.conf
zigbee2mqtt/data/configuration.yaml
```

Docker volumes are rooted at `/mnt/dietpi_userdata/zigbee-mqtt/` on the host
(DietPi), mirroring this same relative layout:

```
/mnt/dietpi_userdata/zigbee-mqtt/mosquitto/config/mosquitto.conf
/mnt/dietpi_userdata/zigbee-mqtt/mosquitto/data/
/mnt/dietpi_userdata/zigbee-mqtt/mosquitto/log/
/mnt/dietpi_userdata/zigbee-mqtt/zigbee2mqtt/data/configuration.yaml
```

## Setup

1. Copy `mosquitto/config/mosquitto.conf` and
   `zigbee2mqtt/data/configuration.yaml` to the matching paths under
   `/mnt/dietpi_userdata/zigbee-mqtt/` on the Pi.
2. Find the ConBee II's stable device path and update the `devices:` entry
   in `docker-compose.yml` if it changes (currently pinned to serial
   `DE2122355`):
   ```bash
   ls -l /dev/serial/by-id/
   ```
3. Create the Mosquitto password file and set the same password in
   `configuration.yaml`'s `mqtt.password` (replacing `CHANGE_ME`):
   ```bash
   docker run --rm -v "/mnt/dietpi_userdata/zigbee-mqtt/mosquitto/config:/mosquitto/config" \
     eclipse-mosquitto:2 mosquitto_passwd -c -b /mosquitto/config/passwd z2m YOUR_PASSWORD
   ```
4. Start the stack:
   ```bash
   docker compose up -d
   docker compose logs -f zigbee2mqtt
   ```
5. Open the Zigbee2MQTT frontend at `http://<pi-ip>:8085`.

## Pairing devices

Since the network isn't migrated from deCONZ, each sensor/switch needs to be
re-paired:

1. In the frontend, click **Permit join (all)**.
2. Factory-reset the device (procedure varies by model — check
   [zigbee2mqtt.io/supported-devices](https://www.zigbee2mqtt.io/supported-devices/)).
3. Confirm it appears and completes its interview in the frontend/log, then
   rename it.
4. Repeat one device at a time, then disable permit join.

## Notes

- ConBee II has no official EmberZNet firmware (that only exists for ConBee
  III / newer Silicon Labs sticks), so it stays on its deCONZ firmware and
  uses Zigbee2MQTT's `deconz` adapter driver.
- Migrating the deCONZ network itself (to skip re-pairing) isn't officially
  supported by Zigbee2MQTT and proved unreliable in practice, so this stack
  starts a fresh network instead.
