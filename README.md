# friend-dispenser

Standalone Wi-Fi desk button (ESP32 + ESPHome) that unlocks a door managed by
**UniFi Access**. Press the button → the ESP32 fires an HTTPS request straight to
the UniFi NVR's Access API → the door relay releases. No Home Assistant, no cloud,
no hub in the middle — the chip talks to the controller directly.

---

## Hardware

| Part | Detail |
|------|--------|
| MCU | ESP32 dev board (`board: esp32dev`), USB-serial bridge (`/dev/cu.usbserial-*` on macOS) |
| Button | Momentary push button wired **GPIO4 → button → GND** |
| Input mode | `INPUT_PULLUP`, `inverted: true` (idle HIGH, press pulls to GND) |
| Debounce | `delayed_on: 20ms` software filter |

The board enumerates as a CP210x/CH340 USB serial device. On macOS it showed up as
`/dev/cu.usbserial-1140` — the suffix can change between machines/ports, so list with
`ls /dev/cu.usbserial-*` before flashing over USB.

## Network topology

| Node | Address |
|------|---------|
| Gateway / router | `<gateway>` |
| ESP32 (static) | `<static_ip>` (subnet `<subnet>`, GW `<gateway>`) |
| UniFi NVR (runs the Access app) | `<nvr_ip>` |
| Access API port | `12445` (HTTPS, self-signed cert) |

The example above puts the button and NVR on different subnets. If yours are split by a
VLAN/firewall, make sure the button's subnet can reach `<nvr_ip>:12445` — that's the
first thing to check if unlocking breaks:

```bash
nc -vz -w3 <nvr_ip> 12445          # TCP reachability
curl -sk https://<nvr_ip>:12445/api/v1/developer/doors \
  -H "Authorization: Bearer <TOKEN>"      # read-only, returns 200 if token+route OK
```

## UniFi Access API

- **Door UUID:** `<door_uuid>`
- **Unlock endpoint:** `PUT https://<nvr_ip>:12445/api/v1/developer/doors/<door_uuid>/unlock`
- **Auth:** `Authorization: Bearer <token>` header. (This controller rejects `X-API-KEY`.)
- **Success response:** `200` + `{"code":"SUCCESS","data":"success","msg":"success"}` (51-byte body).
- **Body:** none required.

### ⚠️ The token must be WRITE-capable
This is the single biggest gotcha. A **read-only** token will happily
`GET /doors` (200) but returns **`401 CODE_UNAUTHORIZED`** on unlock. Mint the token from
an **Owner / Full-Management admin**:

> UniFi Access → **Admins & Users** → click the account → **Create API Token**

Tokens created on the **Integrations** page or **General Access → Advanced** are
read-only and will fail to unlock.

### Find/confirm the door UUID
```bash
curl -sk https://<nvr_ip>:12445/api/v1/developer/doors \
  -H "Authorization: Bearer <TOKEN>" | python3 -m json.tool
# look for the object with "type":"door" — its "id" is the UUID for the /unlock path
```

---

## Repo layout

```
friend-dispenser.yaml   Generic ESPHome firmware logic — committed, no site values
config.yaml             Your deployment config + credentials — GITIGNORED, never committed
config.yaml.example     Template; copy to config.yaml and fill in
.gitignore              ignores config.yaml + build artifacts
LICENSE                 MIT
```

`friend-dispenser.yaml` is fully generic — it pulls every deployment-specific value
(Wi-Fi, IPs, NVR address, door UUID, token, keys) from `config.yaml` via
`substitutions: !include config.yaml`, referenced as `${var}`. Nothing site-specific
or secret lives in the committed files, so the repo is safe to publish.

Keys expected in `config.yaml`: `wifi_ssid`, `wifi_password`, `static_ip`, `gateway`,
`subnet`, `nvr_ip`, `nvr_port`, `door_uuid`, `unlock_auth` (full `"Bearer <token>"`),
`api_key` (native-API encryption), `ota_password`.

## Toolchain setup

ESPHome 2026.6.x. Install once:

```bash
pipx install esphome         # or: python3 -m venv venv && venv/bin/pip install esphome
```

Then from the repo root:

```bash
cp config.yaml.example config.yaml     # first time only, then fill in real values
esphome config friend-dispenser.yaml          # validate
```

## Flashing

**Over USB** (first flash, or if OTA is broken):
```bash
esphome run friend-dispenser.yaml --device /dev/cu.usbserial-XXXX
```

**Over the air** (once it's on the network — no USB needed):
```bash
esphome run friend-dispenser.yaml             # discovers <static_ip>, uses ota_password
```

**Live logs:**
```bash
esphome logs friend-dispenser.yaml --device /dev/cu.usbserial-XXXX   # USB
esphome logs friend-dispenser.yaml                                   # over network/API
```

---

## Replacing the ESP32

If the board dies or you swap in a new one:

1. Wire the new board the same way: button on **GPIO4 → GND**.
2. Plug in USB, find the port: `ls /dev/cu.usbserial-*`.
3. Ensure `config.yaml` exists (copy from `config.yaml.example` if on a fresh machine).
4. Flash over USB: `esphome run friend-dispenser.yaml --device /dev/cu.usbserial-XXXX`.
5. Confirm it joins Wi-Fi and takes the static IP:
   ```bash
   ping <static_ip> && curl -s -o /dev/null -w "%{http_code}\n" http://<static_ip>/
   ```
6. Test the button. First press should return `200` in the logs and release the door relay.

Nothing on the UniFi side needs to change — the door UUID and token are unchanged.
If you also rotate the API token, update `unlock_auth` in `config.yaml` and reflash.

To move the button to a **different door**: set `door_uuid` in `config.yaml`
(get the new UUID from the `/doors` list above). To point at a different controller,
set `nvr_ip` / `nvr_port`.

To move it to a **different network/subnet**: update `wifi_ssid`/`wifi_password` and the
`static_ip`/`gateway`/`subnet` values in `config.yaml`. All of it lives in `config.yaml`
now — `friend-dispenser.yaml` never needs editing for a redeploy.

---

## Troubleshooting

Serial is ground truth. If Wi-Fi/API behavior is unclear, watch the logs while pressing.

| Symptom | Cause | Fix |
|---------|-------|-----|
| No serial output at all; `invalid header: 0xffffffff` + `RTCWDT_RTC_RESET` boot loop | Blank/erased flash — no firmware | Reflash over USB |
| `4-Way Handshake Timeout` on every AP, strong signal | Wrong Wi-Fi password (PSK) | Fix `wifi_password` |
| Associates but never gets IP; UniFi WLAN is WPA3-only or PMF=Required | ESP32 handshake incompat | Set WLAN to WPA2 (or mixed), PMF=Optional |
| Unlock returns **404** | Wrong method/route — using `POST` (or bad UUID) | Must be `PUT`; verify UUID via `/doors` |
| Unlock returns **401 `CODE_UNAUTHORIZED`** | Read-only API token | Mint a write-capable token (see above) |
| Button fires unlock repeatedly (~1/sec) while pressed | Pin bounce OR button held; also see rate-limit note | Check GPIO4 wiring; 5s rate-limit caps it to 1 unlock/5s |

### Reading serial without ESPHome (raw)
```bash
stty -f /dev/cu.usbserial-XXXX 115200 raw
# then read the device, e.g. with a short pyserial script; toggling DTR/RTS resets the chip
```

## Behavior notes / known issues

- **Rate limit:** the unlock is wrapped in an ESPHome `script` with `mode: single` and a
  trailing `delay: 5s`. Any press during that window is dropped (logs
  `Script 'unlock_door' is already running!`). Net effect: **max one unlock per 5s**.
  Change the `delay:` to retune.
- **TLS:** `verify_ssl: false` because the NVR uses a self-signed cert. The `esp-idf`
  framework (not Arduino) is used deliberately — it handles the HTTPS/TLS heap better.
- **Security:** native API is encrypted (`api_key`) and OTA is password-protected
  (`ota_password`), so a LAN peer can't drive or reflash the device. The web dashboard on
  `:80` is unauthenticated but only exposes a read-only `binary_sensor` — it cannot trigger
  the unlock (only a physical press runs `on_press`).

---

## License

[MIT](LICENSE) © 2026 Trevor Ellermann
