# Mi Door and Window Sensor 2 (MCCGQ02HL)

A Xiaomi **MCCGQ02HL** contact sensor. Researched **2026-08-22**; **not yet integrated.**

> **TL;DR** — it's **BLE, not Zigbee**, so it will never join the Tube gateway's Zigbee mesh.
> But it needs **no Xiaomi gateway either**: HA's native `xiaomi_ble` integration reads it over
> the BLE proxy we already run. The one requirement is a **bindkey**, and unlike the
> thermometers that key is needed **permanently at runtime**.

## ⚠️ The naming trap

| Device | Radio |
|---|---|
| Mi / Aqara Door & Window Sensor (`MCCGQ01LM` / `MCCGQ11LM`) | **Zigbee** |
| Mi Door & Window Sensor **2** (`MCCGQ02HL`) — **ours** | **BLE** |

The "2" moved to Bluetooth LE. Anyone reasoning from the older model's Zigbee support will get
this wrong. It cannot be paired to ZHA.

## What it is

| | |
|---|---|
| Model | `MCCGQ02HL` |
| Radio | **BLE** (encrypted Xiaomi MiBeacon advertisements) |
| Battery | CR2032 |
| Extras | **built-in illuminance sensor** as well as the contact reed |
| Integration | HA native **`xiaomi_ble`** |
| Custom firmware | ❌ **none available** — see below |

## How it reaches HA

Exactly the path already built for the [LYWSD03MMC thermometers](../xiaomi-lywsd03mmc/):

```
MCCGQ02HL (BLE)
  └─ Tube gw `Esp_Bluetooth` → ESPHome bluetooth_proxy
       └─ HA native `bluetooth` → xiaomi_ble integration
```

No Xiaomi hub, no cloud, no Hub 2. HA should discover it and then **prompt for a bind key**.

### The bindkey is mandatory — and permanent

Xiaomi encrypts these advertisements. Without the key you get a device that reports
**RSSI and nothing else** — that is by far the most common complaint about this sensor in the HA
forums, and it is always the missing key.

We have already done this procedure once: bind the device to Mi Home (any account), then extract
the key with
[`Xiaomi-cloud-tokens-extractor`](https://github.com/PiotrMachowski/Xiaomi-cloud-tokens-extractor).
The existing keys live in the Bitwarden item **"Xiaomi Mi Home — device tokens & bind keys
(homelab)"** — **add this device's key there too.**

> 🛑 **Important difference from the thermometers.** For the LYWSD03MMC the bindkey was only
> needed to *authorise the custom flash*, and became unused once they ran pvvx/BTHome. Here the
> device stays on **stock firmware**, so the bindkey is **load-bearing forever**. Losing it means
> re-registering the device with Mi Home to extract it again.

## No custom-firmware escape hatch

There is **no pvvx firmware for the MCCGQ02HL**. It has been
[requested and discussed](https://github.com/pvvx/ATC_MiThermometer/discussions/541) but never
delivered, so the BTHome conversion we used on the thermometers is not available. Stock +
bindkey is the only path.

Alternative if we'd rather decode it on an ESP node instead of in HA:
[`Fabian-Schmidt/esphome-xiaomi_mccgq02hl`](https://github.com/Fabian-Schmidt/esphome-xiaomi_mccgq02hl).
Same bindkey requirement. **Prefer native `xiaomi_ble`** — one less moving part, and it needs no
ESP board dedicated to it.

## Good news: no reporting-interval problem

Unlike every temperature sensor in [`../../sensor-reporting.md`](../../sensor-reporting.md), this
is **event-driven** — it advertises the instant the magnet separates, so latency is sub-second
with nothing to tune. No `conf_report`, no `EF00` cluster, no firmware-fixed Matter interval.

## Watch-outs

- **Another BLE advertiser on the `Esp_Bluetooth` path.** The recorder exclusion for
  `esphome.on_ble_advertise` already contains the
  [flood problem](../tube-zb-gw-efr32/#-esp_bluetooth-switch--ble-gateway-mode--it-can-take-ha-down),
  and the decoded entity states arrive via the normal `bluetooth` path and are still recorded —
  which is what we want. But this does add load to a path that has taken HA down twice; the
  dedicated ESP32 `bluetooth_proxy` ([#251](https://github.com/Chrison-Homelab/Homelab/issues/251))
  remains the durable relay.
- **BLE range.** Coin-cell BLE at low TX power on an exterior door may sit at the edge of the
  proxy's range — worth checking RSSI once placed, and a reason the dedicated proxy build matters.

## Steps to integrate

1. Register the device in **Mi Home** once; extract token + bindkey with the cloud-tokens extractor.
2. **Save the bindkey to the Bitwarden item** above.
3. Confirm the Tube gw `Esp_Bluetooth` proxy is on and HA sees BLE.
4. Accept HA's `xiaomi_ble` discovery prompt; paste the bindkey.
5. Verify **contact + illuminance + battery** entities appear (not just RSSI).
6. Add the row to [`../../devices.md`](../../devices.md) with the BLE MAC and room.

## Related

- [`../xiaomi-lywsd03mmc/`](../xiaomi-lywsd03mmc/) — the BLE proxy path and the Mi bindkey workflow
- [`../tube-zb-gw-efr32/`](../tube-zb-gw-efr32/) — the gateway providing the BLE proxy, and its flood history
- [`../../sensor-reporting.md`](../../sensor-reporting.md) — why the *temperature* sensors need tuning and this doesn't
