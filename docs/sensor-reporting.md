# Sensor reporting — why the temp/humidity sensors are slow

Researched **2026-08-22**. The complaint that started this: the cheap Tuya temp/humidity
sensors update far too slowly to be useful. This is the *why*, and what actually fixes it.

> **TL;DR** — it isn't that they're cheap. They're slow because they report through Tuya's
> proprietary `EF00` cluster, which **ignores standard `configure_reporting` outright**. No
> amount of tuning fixes a `TS0601`. The fix is devices that expose **standard ZCL clusters**,
> which ZHA can then be told to report faster. And the ceiling on "faster" is not the radio —
> **it's the battery**.

## The root cause

| | Tuya `TS0601` (`EF00`) | Standard ZCL device |
|---|---|---|
| Temp/humidity delivered via | proprietary `EF00` cluster | `0x0402` / `0x0405` |
| Honours `configure_reporting` | ❌ **no — silently ignored** | ✅ yes |
| Fastest achievable | whatever the firmware decided | ~20-30 s floor |

Even the Tuya models that *do* expose a report-interval setting (e.g. `TH01Z`) cap out at a
**5-minute minimum**. So the requirement isn't "a better sensor" — it's **a device whose
reporting mechanism is actually reachable**.

## The Environment Sensor T1 is a ZG-227Z — and it is flashable

The **Environment Sensor T1** in Karl's bedroom (see [`devices.md`](devices.md)) is recorded as
Tuya `TS0601` / `_TZE200_a8sdabtg`. That identifier pair resolves to the **ZG-227Z**
(and the LCD variant **ZG-227ZL**), and per [pvvx's teardown](https://pvvx.github.io/ZG-227Z/)
it is:

| | |
|---|---|
| MCU | **TLSR8253** — a Telink TLSR825x, i.e. **[ZigbeeTLc](https://github.com/pvvx/ZigbeeTLc)-capable** |
| Sensor | **AHT20** (software I2C) |
| Power | **CR2450** (hw v3.0) *or* **2×AAA** (hw v1.2) — check which one ours is |
| OTA identifiers | manufacturer code `0x1286`, image type `0x0203` ([ZigbeeTLc#161](https://github.com/pvvx/ZigbeeTLc/issues/161)) |

**So the sensor we already own is convertible.** ZigbeeTLc replaces the `EF00` firmware with
standard ZCL clusters, gives a **3-255 s measurement interval** (default 10 s) and two-decimal
resolution, and pvvx ships a **stock-Tuya → ZigbeeTLc conversion image applied over Zigbee OTA**
— no soldering, no debugger.

> 🛑 **Do not flash this unit first.** `sensor.temperaturer_t1_*` is the **`secondary` input to
> Karl's night heating** and is watched by the sensor watchdog. A failed conversion takes out
> heating. Get a second sensor in place first, prove the flash on that, and only then decide
> whether the T1 is worth converting at all.

Second caveat: if ours is the **CR2450** revision, converting it buys resolution but *not*
sustained fast reporting — see [the battery wall](#the-battery-wall) below.

## The battery wall

This is the part that decides everything, and it gets left out of every blog post. **A coin
cell cannot sustain fast Zigbee reporting.**

- LYWSD03MMC on ZigbeeTLc at **300 s** reporting: 100% → 80% **in one week**; several users
  report [batteries flat in 10-12 days](https://github.com/pvvx/ATC_MiThermometer/issues/742).
- A half-discharged CR2032 can't deliver more than **+3 dBm** for TX, which is why pvvx caps
  radio power at +2 dBm on those devices.

A Zigbee transmission costs far more energy than a BLE advertisement. Practical guidance:

| Target rate | Viable power source |
|---|---|
| ~30-60 s | **2×AAA** — comfortable, months of life |
| ~10 s | **USB power** strongly preferred |
| faster | USB only |
| any rate | coin cell = weeks, not months ❌ |

**Decision (2026-08-22): target 30-60 s on 2×AAA.** That's a large improvement over the
current behaviour and needs no firmware work at all — which makes the whole ZigbeeTLc question
optional rather than load-bearing.

## ZHA has no reporting UI — use zha-toolkit

We're on **ZHA** ([decided 2026-07-18](devices.md#zigbee--staying-on-zha-decided-2026-07-18)),
and ZHA exposes no way to configure reporting from the UI. Install
**[`zha-toolkit`](https://github.com/mdeweerd/zha-toolkit)** from HACS and call it per sensor.

```yaml
# Temperature — cluster 0x0402 (1026), values in 0.01 °C
action: zha_toolkit.conf_report
data:
  ieee: sensor.snzb_02d_temperature   # entity or IEEE both accepted
  cluster: 1026
  attribute: 0
  min_interval: 30      # floor — below ~20 s it doesn't measure faster anyway
  max_interval: 300     # heartbeat, so the entity never goes stale
  reportable_change: 20 # = 0.20 °C
  tries: 100            # ⚠️ load-bearing, see below
  event_done: zha_done
```

```yaml
# Humidity — cluster 0x0405 (1029), values in 0.01 %RH
action: zha_toolkit.conf_report
data:
  ieee: sensor.snzb_02d_humidity
  cluster: 1029
  attribute: 0
  min_interval: 30
  max_interval: 600
  reportable_change: 100  # = 1.00 %RH
  tries: 100
  event_done: zha_done
```

Read that as: *report immediately on a 0.2 °C move, but never more than once per 30 s, and send
a heartbeat every 5 min regardless.* In a stable room it mostly idles at the heartbeat, which is
what keeps AAA life in months.

**Two gotchas that will cost you an evening:**

- **`tries: 100` is not optional.** These are sleepy end devices; a single attempt just times
  out. The service retries until the device wakes.
- **Wake the device while it retries.** On the SONOFF SNZB-02D that means pressing the button
  for **1 second**. **Not 5** — 5 s resets pairing and you'll be re-joining it to the mesh.
- `min_interval` below ~20 s is pointless; the hardware doesn't sample faster.

## What to buy

Prices are AliExpress NZ$ as at 2026-08-22, via `~/marketplace-tools/aliexpress.js`
(saved search `zigbee_th_sensor`).

| Device | Price | Why |
|---|---|---|
| **SONOFF SNZB-02D** | ~NZ$17 | **The pick.** 2×AAA, standard ZCL, `configure_reporting` [confirmed working](https://github.com/Koenkk/zigbee2mqtt/discussions/24229), 2.5" LCD. **SONOFF OTA is enabled by default in ZHA**, so its own firmware stays current for free |
| SONOFF SNZB-02P | ~NZ$15 | Same silicon story, no display |
| Tuya LCD, "battery **or USB**" | ~NZ$10 | The **ZY-ZTH02 / TS0201** form factor: **TLSR8258 + clone SHT30, 2×AAA**, so ZigbeeTLc-flashable, and takes USB if we ever want sub-10 s. Buy **one**, not five — see the lottery warning |
| Xiaomi Mijia Meter 3 (`MJWSD06MMC`) | ~NZ$16 | ZigbeeTLc-supported, OTA-flashable over BLE. Avoids the LYWSD03MMC **HW B1.6** unflashable trap |

**Known limitation of the SNZB-02D:** it will not resolve finer than **0.2 °C** no matter what
`reportable_change` is set to. If we ever want two-decimal precision, that's the argument for
ZigbeeTLc — not speed.

### ⚠️ Buying these on AliExpress is a lottery

**Sellers never print the real model code.** Searching `TH03Z`, `ZG-227Z` or `MHO-C401N`
returns laptop bezels and PLC cables — zero relevant hits. Every listing is generically titled
*"Tuya ZigBee Mini Temperature Humidity Sensor"*. You must search the generic phrasing and
identify the device **from its photo and power source**, and accept that on a no-name listing
**the chip inside changes between batches**. Only branded gear (SONOFF, Xiaomi) is reliably
identifiable. Also: `NZ$1.7` on a card is the "new shopper" teaser, not a price.

## Why not just use BLE

We already run BLE sensors well (pvvx/BTHome LYWSD03MMC ×2 via the gateway proxy — see
[`devices/xiaomi-lywsd03mmc/`](devices/xiaomi-lywsd03mmc/)), and a BLE advert is much cheaper
energy-wise than a Zigbee TX, so BLE is genuinely *better* at high frequency.

The reason to keep new sensors on Zigbee anyway: **every added BLE advertiser feeds the
`Esp_Bluetooth` advert path**, which has taken HA down twice and filled the recorder with 28 M
rows ([details](devices/tube-zb-gw-efr32/#-esp_bluetooth-switch--ble-gateway-mode--it-can-take-ha-down)).
That's contained now via the recorder exclusion, but Zigbee end devices on the ZHA mesh don't
touch that path at all. Zigbee is the cleaner thing to scale.

## Free experiment available

Both LYWSD03MMC units are **HW B1.4** and already pvvx-flashed, so one can be converted
**BLE → ZigbeeTLc Zigbee** through the same browser TelinkMiFlasher already used, at zero cost.
That's the cheapest possible way to see what ZigbeeTLc reporting actually feels like on our ZHA
mesh before spending anything or touching the T1.

Expect **~20-25% shorter battery life** on Zigbee firmware vs BLE, and note that pvvx's
*custom* config attributes (measurement interval, offsets) need a ZHA quirk that
[isn't merged](https://github.com/zigpy/zha-device-handlers/issues/4148) — temp/humidity/battery
work fine, and intervals get set with `conf_report` above, same as any other device.

## Related

- [`devices.md`](devices.md) — the inventory, incl. the Environment Sensor T1 row
- [`devices/xiaomi-lywsd03mmc/`](devices/xiaomi-lywsd03mmc/) — the BLE thermometers and the pvvx flashing workflow
- [`thread-and-matter.md`](thread-and-matter.md) — why Matter/Thread sensors do **not** solve this
- `~/marketplace-tools/` — `aliexpress.js` + the `zigbee_th_sensor` saved search
