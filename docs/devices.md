# Smart-home device inventory

Smart-home end-devices. **Seeded 2026-07-17** from UniFi (network-visible clients) and
**enriched 2026-07-18** from the Home Assistant device registry (which also surfaces
Zigbee / cloud / IR devices that never hold a Wi-Fi lease). Annotate the human bits
(exact room, purchase) as you go.

> Sources: UniFi = Wi-Fi/wired clients (gives VLAN·IP·MAC). HA = manufacturer/model/room
> + integration. Battery-BLE devices (e.g. the Xiaomi thermometers) appear in neither until
> a BLE proxy exists. IPs for Wi-Fi Tuya/Tapo devices are `_TODO_` (find in UniFi).

> ⚠️ **Post-move (2026):** the HA config carried over from the **previous rental**, so its
> device registry contains **ghosts** — devices that didn't make the move or aren't set up
> yet in the new home. **Rule of thumb:** a device with a *current UniFi lease* is confirmed
> live; an **HA-only** entry (no lease, and not Zigbee/BLE/IR) may be **stale — verify
> physically.** Rows below tagged _verify_ are HA-only and unconfirmed in the new home.

> ✅ **The ghost problem is largely gone as of 2026-08-09.** CT 6005 was built **from scratch**
> with **no backup restored** (see [`../homeassistant.lxc.yaml`](../homeassistant.lxc.yaml)), so its
> registry contains only what was deliberately re-added — the old instance's 530-of-701 unavailable
> entities did not come across. That changes how to read this table: on CT 6005, an entity that is
> `unavailable` is a **real, current fault**, not a leftover. The 2026-08-11 Tuya audit below is the
> first pass done on that basis.

## Devices

| Device | Type | Room | Integration | VLAN · IP | MAC | Model / notes | Purchased |
|--------|------|------|-------------|-----------|-----|---------------|-----------|
| Yeelight lights | Lights | _TODO_ | Yeelight / `xiaomi_home` | IoT (Lamp4 = `10.40.18.24`) | `58:b6:23:47:0e:79` (Lamp4) | ⚠️ **verify** — HA lists 7 (`lamp4`/`color3`/`color5`/`mono6`/`monoa`) but **most didn't survive the move**; confirm which are actually alive | _owned (partial)_ |
| AC — Master Bedroom (upstairs) | Air conditioner | Master Bedroom | planned — **wired CNS** (MHI-AC-Ctrl-ESPHome) *or* IR fallback | _(CNS serial / or IR)_ | — | **Mitsubishi Heavy Industries (MHI) SRK-ZSA "Bronte"** R32 inverter — *not* Mitsubishi Electric (remote **RLA502A720** → SRK20/25/35/50ZSA-W family; exact kW **TBC** from the outdoor `SRC…` nameplate). Wired route = CNS/SPI, not Daikin S21 or Mits-Electric CN105. **Not set up yet** | _owned_ |
| AC — Downstairs | Air conditioner | Downstairs | **planned — S21 wired** (XIAO ESP32-C6 + ESPHome) ([docs →](devices/daikin-ftxf-s21/)) | IoT 1040 · `_TODO_` (C6, Wi-Fi) | — | **Daikin FTXF50TVMA** (Cora, 5.0 kW; ser. `E009281`). **Not IR** — taps the S21 port for local bidirectional control | _owned_ |
| Sony Bravia KD-49XF7596 | TV / cast | Lounge | `braviatv` + Cast (→ AirCast CT 6002) | legacy · `192.168.179.132` | `d8:9c:67:cb:42:eb` | Sony Android TV (Chromecast built-in); one device, multiple integrations — **move to IoT 1040** | _owned_ |
| Philips Android TV | TV / cast | Master Bedroom | Android TV / Cast | legacy · `192.168.178.208` | `84:3e:1d:6c:bf:a2` | **TPV / TP Vision** "2020 FHD Android TV" (= Philips) — OUI Gaoshengda ODM | _owned_ |
| Apple TV 4K (gen 3) | Media / HomeKit-Matter hub | Master Bedroom | `apple_tv` | legacy · `192.168.178.71` | `c4:f7:c1:51:61:76` | Apple | _owned_ |
| Echo · Lounge | Voice assistant | Lounge | `alexa_media` | legacy · `192.168.178.200` | `e8:4c:4a:40:0c:52` | Amazon Echo | _owned_ |
| Echo · Bedroom | Voice assistant | Bedroom | `alexa_media` | legacy · `192.168.178.145` | `10:ce:02:d0:d3:99` | Amazon Echo | _owned_ |
| Broadlink RM4 mini | IR blaster | _TODO_ | `broadlink` (+ SmartIR) | _TODO_ | — | ⚠️ **not installed yet** — needed to control both ACs; the old Haier AC (`climate.lounge_ac`) was the **previous rental** and is gone | _owned_ |
| Heater | Smart heater | Karls Bedroom | Tuya cloud (`tuya`) | IoT · `_TODO_` | — | Tuya `PEH224/225HA` — ✅ **live** (`climate.heater`, `switch.heater_child_lock`). **Factory-reset + re-paired 2026-08-11**; the Tuya device id survived (`ebe662f772153847dasmus`), so the entity ids and Karl's heating automation needed no rewiring. Load-bearing: see [Tuya](#tuya--cloud-only-and-half-the-fleet-is-dark) | _owned_ |
| Tower Fan ("Bedroom Fan") | Fan | Karls Bedroom | Tuya cloud (`tuya`) | IoT · `_TODO_` | — | Goldair Platinum Tower Fan (`fan.bedroom_fan`, + `sensor.bedroom_fan_temperature`) — ⛔ **offline** since 2026-08-09; has never reported to CT 6005 | _owned_ |
| Standing Fan | Fan | Office | Tuya cloud (`tuya`) | IoT · `_TODO_` | — | Tuya (model TBC; `fan.standing_fan`) — ⛔ **offline** since 2026-08-09 | _owned_ |
| Aromalife | Aroma diffuser | Karls Bedroom | Tuya cloud (`tuya`) | IoT · `_TODO_` | — | Tuya "Aromalife" — ✅ **live** (`switch.aromalife_power`, `switch.aromalife_spray`) | _owned_ |
| Smart Plug · Lounge | Smart plug | Lounge | Tuya cloud (`tuya`) | IoT · `_TODO_` | — | Tuya "smart plug" with energy metering (`switch.smart_plug_lounge_socket_1` + current/power/voltage/total-energy) — ⛔ **offline** since 2026-08-09. **Was missing from this table** until the 2026-08-11 audit | _owned_ |
| Smart Plug · Guest Room | Smart plug | Guest Room | Tuya cloud (`tuya`) | IoT · `_TODO_` | — | Tuya smart plug, energy metering + child lock + power-on-behaviour (`switch.smart_plug_guest_room_socket_1`) — ⛔ **offline** since 2026-08-09. **Was missing from this table** until the 2026-08-11 audit | _owned_ |
| Alen BreatheSmart 45i | Air purifier | _TODO_ | _TODO_ (Tuya?) | IoT · `10.40.169.147` | `c4:82:e1:6d:15:cc` | Alen 45i True HEPA (live on network) | _owned_ |
| Tapo C200 | Wi-Fi camera | Karls Bedroom | `tapo` (TP-Link) | IoT · `_TODO_` | — | TP-Link Tapo C200 (motion/person/baby-cry) — ⚠️ _verify_ | _owned_ |
| Environment Sensor T1 | **Zigbee** temp/humidity | Karls Bedroom | **ZHA** (via Tube gw) | _(Zigbee, not IP)_ | — | Tuya `TS0601` (`_TZE200_a8sdabtg`); `sensor.temperaturer_t1_*`. ✅ **live on CT 6005** (battery 100%) and **load-bearing** — it is the `secondary` input to Karl's night heating, so it is now watched by the sensor watchdog rather than excluded as dead | _owned_ |
| Nest Protect | Smoke/CO alarm | _TODO_ | _TODO_ (Nest) | IoT · `10.40.2.131` | `d8:c8:0c:b0:9e:28` | Nest Protect | _owned_ |
| Meross Smart Plug | Smart plug | Garage | `meross_lan` (local) | IoT · `10.40.170.241` | `48:e1:e9:dd:88:d9` | Meross `mss310` | _owned_ |
| Samsung Washer | Appliance | Laundry | SmartThings | IoT · `10.40.84.50` | `50:fd:d5:85:6c:92` | Samsung `DA_WM_TP2_20` | _owned_ |
| Xiaomi Temp & Humidity Monitor 2 ×2 | BLE temp/humidity | _TODO_ | ✅ **pvvx `ATC_v58` → BTHome v2** (unencrypted) → HA via Tube-gw BLE proxy ([done →](devices/xiaomi-lywsd03mmc/)) | _not IP (BLE, battery)_ | `A4:C1:38:1F:09:C0`, `A4:C1:38:20:6E:6B` | `LYWSD03MMC` **HW B1.4**; flashed 2026-07-20 (`ATC_1F09C0` / `ATC_206E6B`); Mi keys in BW | _owned (2026-07-18)_ |
| **Tube's ZB Gateway** | Zigbee coordinator **+ BLE proxy (ON)** | Garage | **ZHA** today (`socket://…:6638`); → Z2M planned ([docs →](devices/tube-zb-gw-efr32/)) | legacy · `192.168.179.222` → **move to IoT 1040** | `20:43:a8:c7:62:b3` | OEM "藏机/Cangji" TubesZB `efr32-MGM210-poe` clone; web login `cangji`/`cangji` (in BW); ⚠️ `Esp_Bluetooth` **ON since 2026-07-20** as the LYWSD03MMC BLE proxy — *prev flooded HA ×2, monitor* | _owned_ |
| Xiaomi Smart Home Hub 2 | Multi-protocol hub | _TODO_ | _TODO_ — Zigbee/BLE/IR ([notes →](devices/xiaomi-smart-home-hub-2/)) | **⚠️ detect** (Wi-Fi) | _TODO_ | Alt Zigbee/BLE hub; not the local-first BLE relay | _owned_ |
| Leapmotor C10 | EV (telemetry) | Driveway | Leapmotor Mate (CT 4100) → MQTT → HA | _(cloud API)_ | — | VIN `LFZ93AN93SD112595`; incl. Digital Key | _owned_ |
| **Arrowhead ESL-2** | Alarm panel | whole-house | **RE in progress** — keypad-bus → ESP32 → MQTT → HA ([docs →](devices/arrowhead-esl-2/)) | _not IP (serial/keypad bus)_ | — | ELITE-S; **discontinued/abandonware** | _owned_ |
| **Doorworks GDC6** | Garage-door opener | Garage | **build in progress** — dry-contact + reed → ESP8266/ESPHome → HA `cover` ([docs →](devices/doorworks-gdc6/)) | IoT 1040 (planned) | — | Dumb opener (no cloud/API); DIY ESP fleet build | _owned_ |

> **Not smart-home devices** (seen in HA/UniFi but infra): the "AC LR (…)" entries are
> **UniFi U7LR access points** (not air-con), "USW Flex Mini" are UniFi switches, plus the
> Synology NAS, Proxmox nodes, and weather/rubbish-collection service integrations.

## Tuya — cloud-only, and half the fleet is dark

Audited **2026-08-11** against CT 6005's registries and recorder. The `tuya` config entry
(account `christian.simon1988@gmail.com`) is healthy and holds **six** devices — two more than
this table used to list. Their state is not uniform, and the split matters:

| Device | Entity | State |
|---|---|---|
| Heater (Karls Bedroom) | `climate.heater` | ✅ live — re-paired 2026-08-11 |
| Aromalife (Karls Bedroom) | `switch.aromalife_power` | ✅ live |
| Tower Fan (Karls Bedroom) | `fan.bedroom_fan` | ⛔ offline since the 08-09 rebuild |
| Standing Fan (Office) | `fan.standing_fan` | ⛔ offline since the 08-09 rebuild |
| Smart Plug (Lounge) | `switch.smart_plug_lounge_socket_1` | ⛔ offline since the 08-09 rebuild |
| Smart Plug (Guest Room) | `switch.smart_plug_guest_room_socket_1` | ⛔ offline since the 08-09 rebuild |

**The four dark ones are not an HA problem.** They are registered in the Tuya cloud and HA created
their entities correctly at first boot; they have simply never reported since, i.e. they are not
reaching Tuya's cloud at all. Two of the six work from the same account and the same integration,
which rules out credentials. Check power and Wi-Fi on the devices themselves — likely candidates
are units that never came back after the move, or that are still joined to an SSID that no longer
exists. Until then no amount of HA-side work will surface them.

**Every one of these is CLOUD-CONTROLLED, and that has a real cost.** Commands leave the house:
`tuya_sharing` POSTs to `/v1.1/m/thing/<id>/commands`, so each service call is a round trip to
Tuya's API. This is why Karl's night heating was rewritten to **send only on change** on
2026-08-11 — the old shape re-asserted `climate.set_hvac_mode` on every evaluation (~4,949 runs
and ~3,200 cloud calls per day, because the BTHome sensors it triggers on report every ~19 s), and
the API answered often enough with `RemoteDisconnected` to leave error bursts in the log on
08-09/08-10. Anything new that drives a Tuya entity on a timer or a sensor trigger must carry the
same guard.

Measured after the rewrite: **30 automation runs in 9.1 minutes and zero `climate.*` service
calls**, because the heater was already in the state the automation wanted. The automation still
evaluates on every sensor report — that part is deliberate, it is what keeps the room responsive —
it just no longer talks to Tuya unless something has to change. `climate.heater` was also added to
its triggers (mode changes only), so an out-of-band change such as someone pressing the physical
button is corrected within seconds rather than relying on the old ~17-second re-assert to mask it.

### Seasonal devices — kept, but only shown when they are actually there

Three of these are seasonal rather than broken: **both fans live in a cupboard over winter** and the
**heater goes away for summer**. They are deliberately **not** deleted from HA — that keeps their
history, their automations and (for the heater) a device id that has already survived one factory
reset. What was wanted instead was for them to vanish from the dashboard while they are away and
reappear on their own.

Done with native per-card **`visibility:`** conditions on a curated YAML dashboard
(`/config/dashboards/home.yaml`, served at `/smart-home`, registered from `configuration.yaml`):

```yaml
- type: tile
  entity: fan.bedroom_fan
  visibility:
    - condition: state
      entity: fan.bedroom_fan
      state_not: unavailable
```

No HACS, no helper entities, no automation to maintain — the card is simply absent while the device
is, and returns within seconds of it reporting again. The **Office section carries the condition at
section level** because the standing fan is its only occupant; without that, an empty "Office"
heading would sit there all winter.

The auto-generated **Overview is deliberately left alone** as the catch-all, so a newly added device
is never invisible just because nobody hand-added it to the curated view.

> ⚠️ **Hiding an offline device also hides a fault** — fine for a fan in a cupboard, not fine for
> the heater in a toddler's room. So the heater's disappearance is **also alerted on**: the sensor
> watchdog pushes to the phone when `climate.heater` is unavailable *and heat is actually wanted*
> (inside the 18:30–07:00 window with the room at or below the same 17 °C floor the heating
> automation acts on). That is season-agnostic on purpose — in summer the room is never that cold at
> night, so there is no monthly switch to remember to flip, and a heater that dies in July still
> raises an alarm. Verified by simulation both ways: cold room + missing heater alerts, warm room +
> missing heater stays silent.

> **`localtuya` is the standing escape hatch, not adopted.** Local control would remove the cloud
> dependency for the heater entirely, which is attractive for a device a toddler's room depends on.
> It needs per-device local keys, so it is a deliberate project rather than a quick swap — and note
> the heater's ID has now survived one factory reset, so the keys would likely be stable.

## Zigbee — staying on ZHA (decided 2026-07-18)

The Zigbee mesh runs on **ZHA** (native HA) against the standalone Tube gateway coordinator
(`socket://192.168.179.222:6638`), and **we're keeping it that way** — no Zigbee2MQTT migration
(it'd force re-pairing every device for no real gain; the coordinator is a standalone box either
way). **→ The planned Zigbee2MQTT LXC (CT 6004) is dropped.** The Tube gateway stays as the ZHA
coordinator; the XZG firmware move (#259) is still compatible (same socket). ESPHome LXC (6003,
#251) is unaffected — it's for the DIY ESP fleet.

## Per-device deep-dives

Devices with their own reverse-engineering / integration notes live under [`devices/`](devices/):

- [`daikin-ftxf-s21/`](devices/daikin-ftxf-s21/) — Downstairs Daikin FTXF50 (Cora): local, bidirectional HA climate via the **S21 port** (XIAO ESP32-C6 + ESPHome) instead of IR — incl. wiring, [BOM + level-shifter](devices/daikin-ftxf-s21/hardware/) and a ready [ESPHome config](devices/daikin-ftxf-s21/esphome-daikin-downstairs.yaml.example).
- [`arrowhead-esl-2/`](devices/arrowhead-esl-2/) — Arrowhead ESL-2 alarm panel: local HA integration via keypad-bus tap (ESP32), incl. the full [RS232 protocol reference](devices/arrowhead-esl-2/rs232-protocol.md).
- [`doorworks-gdc6/`](devices/doorworks-gdc6/) — Doorworks GDC6 garage-door opener: dumb-opener → HA `cover` via ESP8266/ESPHome (dry-contact trigger + reed-switch state); staged build plan.
- [`tube-zb-gw-efr32/`](devices/tube-zb-gw-efr32/) — TubesZB Zigbee coordinator (ESP32 + EFR32, PoE): access/handbook, the OEM-clone finding, the `Esp_Bluetooth` HA-flood incident, and the Z2M/XZG plans.
- [`xiaomi-lywsd03mmc/`](devices/xiaomi-lywsd03mmc/) — Xiaomi LYWSD03MMC BLE thermometers: the "HA has no Bluetooth" root cause, pvvx/BTHome flashing, HW-B1.6 caveat, and BLE-proxy relay options.
- [`xiaomi-smart-home-hub-2/`](devices/xiaomi-smart-home-hub-2/) — Xiaomi Hub 2 (Zigbee/BLE/IR): integration options + why it isn't the local-first BLE relay yet.

## To identify / tidy
- **Wi-Fi IPs** for the Tuya/Tapo/Broadlink devices — capture from UniFi + DHCP-reserve.
- **The four dark Tuya devices** (both fans, both smart plugs) — check power/Wi-Fi at the device;
  they have not reached the Tuya cloud once since 2026-08-09 ([why that is device-side](#tuya--cloud-only-and-half-the-fleet-is-dark)).
- **Tube's ZB Gateway** (`192.168.179.222`) — on the **legacy** net; move to **IoT 1040** + DHCP-reserve before any Z2M cutover (see [deep-dive](devices/tube-zb-gw-efr32/)).
- **Consumer/media devices on the legacy net** (Echoes, Apple TV, Philips TV, Sony TV) — still on legacy `192.168.178/9`, not segmented onto **Consumer 1020** or **IoT 1040**. Tidy-up candidate.
- **Xiaomi Hub 2** — power on + find on UniFi (Wi-Fi), DHCP-reserve, record model/MAC ([notes](devices/xiaomi-smart-home-hub-2/)).
- _Resolved 2026-07-20:_ ~~**Xiaomi thermometers** — check HW revision~~ → **B1.4**, both flashed **pvvx `ATC_v58` → BTHome v2** and in HA ([done](devices/xiaomi-lywsd03mmc/)).
- **BLE into HA — interim-solved 2026-07-20.** HAOS has no Bluetooth radio; the thermometers now
  reach HA via the **Tube gw's `Esp_Bluetooth` BLE-proxy** — re-enabled and working, but ⚠️ this mode
  **flooded HA offline twice before** ([details](devices/tube-zb-gw-efr32/#-esp_bluetooth-switch--ble-gateway-mode--it-can-take-ha-down)),
  so **monitor it**. The durable relay is still a **dedicated ESP32/ESPHome `bluetooth_proxy`**
  (see [`esp-fleet.md`](esp-fleet.md), #251) — the graduation path if the gateway proxy misbehaves.
- _Resolved 2026-07-18:_ ~~"Tuya device" `10.40.169.147`~~ → Alen air purifier; ~~"Unknown" `10.40.2.131`~~ → Nest Protect.

## Related docs
- [`esp-fleet.md`](esp-fleet.md) — bare ESP boards on hand for DIY builds (BLE proxy, ESL-2, sensors) + which chip does what.

## Notes
- **Integrations at a glance:** Zigbee → **ZHA** (via Tube gw); Matter → matter-server (CT 6001);
  MQTT → broker (CT 6000); Chromecast → AirCast (CT 6002); the C10 → Mate (CT 4100); Tuya/Meross/
  SmartThings/Tapo/Alexa/Nest → their own integrations; IR → Broadlink RM4 mini + SmartIR (**planned** — blaster not installed yet). **ACs split by brand:** the **downstairs Daikin** goes **wired S21 + ESPHome** (not IR — [docs](devices/daikin-ftxf-s21/)); the **upstairs Mitsubishi Heavy Industries** unit → **wired CNS** (MHI-AC-Ctrl-ESPHome, SPI) or IR fallback (TBD).
- The SmartHome **services** themselves (broker/matter/aircast/HA) live in
  [`../README.md`](../README.md), not here.
</content>
