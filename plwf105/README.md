# PETLIBRO PLWF105 ESPHome Firmware

Local ESPHome firmware for the PETLIBRO Dockstream Smart Fountain (PLWF105). It replaces the PETLIBRO cloud firmware with local control, water-level monitoring, estimated drinking records, Home Assistant integration, and a password-protected web interface.

This project is based on [taylorfinnell/petlibro-esphome](https://github.com/taylorfinnell/petlibro-esphome).

## Features

- Local pump control
- Continuous or timed pump operation
- Water-level percentage and volume estimates
- Estimated last drinking-event volume
- Rolling 24-hour water-consumption estimate
- Empty-tank, removed-tank, and pump-error detection
- Filter-life tracking
- Guided minimum/maximum water-level calibration
- Home Assistant integration through the ESPHome native API
- Password-protected local web interface
- Wireless ESPHome OTA updates
- No PETLIBRO account or cloud connection required

The PLWF105 has no RFID reader, so drinking records represent combined fountain usage and cannot identify individual pets.

## Hardware

- PETLIBRO PLWF105 Dockstream Smart Fountain
- ESP32-C3-WROOM-03 controller
- HX711 load-cell interface
- 4 MB SPI flash
- 2.4 GHz Wi-Fi

The PCB exposes `GND`, `TX`, and `RX` programming connections. Pads labeled `B` and `G` can be shorted during power-up to enter the ESP32-C3 ROM download mode.

## Before Flashing

Back up the complete 4 MB factory flash before replacing the stock firmware. The factory image contains device-specific identifiers and cloud credentials.

```bash
esptool \
  --chip esp32c3 \
  --port /dev/cu.usbserial-A50285BI \
  --baud 19200 \
  --before no-reset \
  --no-stub \
  read-flash 0x0 0x400000 petlibro_factory_dump.bin
```

Verify the resulting size and record its hash:

```bash
wc -c petlibro_factory_dump.bin
shasum -a 256 petlibro_factory_dump.bin
```

The expected size is exactly `4194304` bytes. Keep factory images private.

## Configuration

Create `plwf105/secrets.yaml`:

```yaml
wifi_ssid: "YOUR_WIFI_NAME"
wifi_password: "YOUR_WIFI_PASSWORD"

web_username: "fountain"
web_password: "A_LONG_UNIQUE_PASSWORD"
```

Do not commit `secrets.yaml`. Confirm that the repository ignores it:

```bash
git check-ignore -v plwf105/secrets.yaml
```

If necessary, add this rule to `.gitignore`:

```gitignore
**/secrets.yaml
```

The configuration uses the `America/Phoenix` timezone. Change the `time:` component if the fountain is installed elsewhere.

## Build

Install ESPHome, then validate and compile from the `plwf105` directory:

```bash
esphome config plwf105.yaml
esphome compile plwf105.yaml
```

With ESPHome 2026.7, the generated images are normally located at:

```text
.esphome/build/petlibro-plwf105/build/firmware.factory.bin
.esphome/build/petlibro-plwf105/build/firmware.ota.bin
```

Use `firmware.factory.bin` for the first serial installation. Use OTA for later updates.

## Initial Serial Installation

Use a 3.3 V USB-to-UART adapter:

| Adapter | PLWF105 |
|---|---|
| GND | GND |
| TX | RX |
| RX | TX |

Do not use 5 V TTL or RS-232 signaling. Power the fountain from its normal USB supply rather than relying on the serial adapter for power.

1. Disconnect fountain power.
2. Keep UART wiring short and away from the pump/power coil.
3. Short the PCB pads labeled `B` and `G`.
4. Apply fountain USB power while holding the short.
5. Confirm the top light is steady rather than slowly flashing.
6. Flash the factory image:

```bash
esptool \
  --chip esp32c3 \
  --port /dev/cu.usbserial-A50285BI \
  --baud 19200 \
  --before no-reset \
  --no-stub \
  write-flash \
  --flash-size 4MB \
  0x0 .esphome/build/petlibro-plwf105/build/firmware.factory.bin
```

After verification completes, remove power, remove the `B`–`G` short, and power the fountain normally.

## Local Web Interface

Open the following URL from a device on the same network:

```text
http://petlibro-plwf105.local/
```

If mDNS is unavailable, use the fountain's DHCP-assigned IP address:

```text
http://192.168.x.x/
```

The page groups everyday entities into:

- **Fountain** — pump and interval mode
- **Water** — percentage, volume, last drink, and rolling consumption
- **Status** — empty, removed tank, and filter state
- **Maintenance** — filter reset and calibration

The web interface uses HTTP rather than HTTPS. Keep it on a trusted local network, enable digest authentication, and do not expose it directly to the internet.

## Home Assistant

Home Assistant should discover the fountain automatically through the ESPHome native API.

Go to **Settings → Devices & services** and look under **Discovered**. If it is not discovered, add the ESPHome integration manually using:

```text
petlibro-plwf105.local
```

The default ESPHome API port is `6053`.

The YAML file does not need to be copied into Home Assistant for integration. Copying it into Home Assistant's ESPHome Device Builder is optional and only needed if builds will be managed there.

## Calibration

This firmware uses a minimum/maximum water-range calibration. It is different from PETLIBRO's original base/tank tare routine.

1. Assemble the fountain and fill it to the desired minimum operating level.
2. Open the local web interface.
3. Press **Start Calibration**.
4. Confirm the yellow LED is blinking slowly.
5. Press the physical front button to save the minimum reading.
6. Confirm the yellow LED changes to a fast blink.
7. Fill the fountain to its maximum line.
8. Press the physical front button again to save the maximum reading.
9. Confirm the blinking stops and the web page shows a sensible percentage.

LED meanings during calibration:

| Yellow LED | Meaning |
|---|---|
| Slow blink | Waiting for minimum level |
| Fast blink | Waiting for maximum level |
| Blinking stopped | Calibration complete |
| Solid | Filter-expiration indication |

During normal operation, the front button resets the filter date when **Front Button Resets Filter** is enabled.

## Water-Consumption Estimates

The firmware samples the HX711 scale, filters noisy readings, and compares stable measurements over approximately one minute. A sufficiently large stable decrease is recorded as a drinking event.

The default conversion values are:

```text
379.59 raw scale units per mL
11225 raw scale units per oz
```

Detected decreases between `400` and `67350` raw units are eligible as drinking events. Tank movement, refilling, cleaning, evaporation, or physical contact can create false readings. Treat the values as useful trends, not medical-grade measurements.

Home Assistant can retain long-term history and provide daily, weekly, and monthly graphs. The ESP itself maintains only its current values and rolling 24-hour estimate.

## OTA Updates

After the first serial installation, compile and upload updates over Wi-Fi:

```bash
esphome run plwf105.yaml --device petlibro-plwf105.local
```

To compile without uploading:

```bash
esphome compile plwf105.yaml
```

## Restoring Factory Firmware

To restore a verified full-flash backup, manually enter download mode and write it at offset zero:

```bash
esptool \
  --chip esp32c3 \
  --port /dev/cu.usbserial-A50285BI \
  --baud 19200 \
  --before no-reset \
  --no-stub \
  write-flash \
  --flash-size 4MB \
  0x0 petlibro_factory_dump.bin
```

Only restore a backup whose size and integrity have been verified. A factory image from another fountain may contain different identifiers and credentials and should not be used.

## Troubleshooting

### Serial corruption or digest mismatch

- Disconnect or de-energize the pump coil.
- Keep TX, RX, and GND wires short.
- Route serial wires away from power wiring.
- Use a stable fountain power supply with a shared ground.
- Reduce the serial rate to `19200` or `9600`.
- Use `--no-stub` if the RAM stub is unreliable.

### `.local` does not resolve

- Confirm the client and fountain are on the same LAN.
- Disable VPN or mobile-data routing temporarily.
- Ensure the router permits multicast/mDNS between clients.
- Use the fountain's IP address or configure a DHCP reservation.

### Captive portal warning

`captive_portal:` requires a fallback `wifi.ap:` configuration. Without `wifi.ap:`, normal Wi-Fi, Home Assistant, OTA, and the web interface still work, but the fallback captive portal is unavailable.

## Security

- Never commit `secrets.yaml`.
- Never publish factory dumps, NVS images, serial numbers, or product secrets.
- Use a strong, unique web password.
- Keep the ESPHome web server limited to a trusted LAN or IoT VLAN.
- Do not forward port 80 or port 6053 from the internet.
- Consider enabling ESPHome native API encryption for Home Assistant.

## License and Attribution

This fork derives from [taylorfinnell/petlibro-esphome](https://github.com/taylorfinnell/petlibro-esphome). At the time this README was prepared, the upstream repository did not contain a visible license file. Public source code without an explicit license is not automatically unrestricted for redistribution. Preserve attribution and obtain or clarify the necessary permission before redistributing modified source or firmware.

PETLIBRO is a trademark of its respective owner. This community firmware is not an official PETLIBRO product.
