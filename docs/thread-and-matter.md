# Thread & Matter — state of play, and the border-router decision

Researched **2026-08-22**, triggered by looking at the IKEA **TIMMERFLOTTE** sensor. Records
why we currently cannot use any Matter-over-Thread device, what it would take, and which option
to pick.

> **TL;DR** — we have a **Matter controller** (`matter-server`, CT 6001) but **no Thread border
> router**, and **zero commissioned Matter nodes**. Those are different things and the controller
> does not imply the router. Also: Matter/Thread sensors have **firmware-fixed reporting
> intervals**, so they are *not* a fix for [slow sensor reporting](sensor-reporting.md) — they're
> the same trap in a different protocol.

## Where we actually are

| Piece | Status |
|---|---|
| Matter controller | ✅ `matter-server` CT 6001, host-net, IoT 1040, IPv6 SLAAC |
| Commissioned Matter nodes | **0** — HA↔CT 6001 was a [clean repoint on 2026-08-02](../matter-server.lxc.yaml), both sides empty |
| **Thread border router** | ❌ **none** |
| 802.15.4 silicon on hand | ✅ 1× ESP32-H2 Super Mini, 1× ESP32-H2-DEV-KIT-N4, 2× XIAO ESP32-C6 ([`esp-fleet.md`](esp-fleet.md)) |

So Matter is **entirely unproven here** — the first Thread device we buy exercises the
controller, the border router, IPv6 on VLAN 1040 and mDNS all at once.

## Do we already own a border router? Probably, but check

Consumer hubs we own that *can* be Thread border routers:

| Device | Border router? |
|---|---|
| **Apple TV 4K (gen 3)** — `192.168.178.71` | ⚠️ **depends on the model.** 128 GB **Wi-Fi + Ethernet** (`A2843`) ✅ · 64 GB **Wi-Fi-only** (`A2737`) ❌ **no Thread radio** |
| Apple TV 4K gen 2 (`A2169`) | ✅ (not what we have) |
| **Echo ×2** (Lounge, Bedroom) | ✅ *if* 4th gen — but **the worst option for HA**; Amazon's Thread credential sharing is the most closed of any ecosystem |
| IKEA DIRIGERA | ✅ — sold locally, but a closed-ish fabric |

**→ Action: confirm the Apple TV's model** (Settings → General → About). The inventory records
it only as "gen 3", and that's exactly the ambiguity that decides whether we own a border router
or not.

### ⚠️ Even a working Apple TV border router has a topology problem

The consumer devices are all on the **legacy `192.168.178/9` net** — the Echoes, both TVs and the
Apple TV are flagged as untidied in [`devices.md`](devices.md#to-identify--tidy). But
`matter-server` sits on **IoT 1040**, and [its shape](../matter-server.lxc.yaml) is explicit that
Matter discovery is **multicast on the device L2** with *"no inter-VLAN boundary"*.

A border router on legacy + a Matter controller on 1040 means bridging IPv6 + mDNS across VLANs.
That is the single most likely thing to bite us, and it is a **network** problem, not a Matter
problem. A border router we place **on 1040 ourselves** sidesteps it entirely — which is the
main argument for building/buying rather than leaning on the Apple TV.

## The distinction that decides the hardware: onboard OTBR vs RCP

Our Zigbee coordinator works by exposing the EFR32's UART over TCP and letting a host-side stack
drive it. Thread hardware splits into boxes that copy that, and boxes that run the whole border
router themselves. **This matters because HA is a container (CT 6005), not HAOS — the HAOS
"OpenThread Border Router" add-on is not available to us.**

| Option | Architecture | Needs the OTBR add-on? | Zigbee too? |
|---|---|---|---|
| **GL.iNet GL-S200** (~NZ$87-175) | **OTBR onboard** — true appliance | ❌ no; core integrations only | ✖ |
| **SMLIGHT SLZB-MR1** (~NZ$69-95) | RCP over Ethernet | ✅ (or DIY container) | ✅ **simultaneously**, separate radios |
| Sonoff Dongle Max | RCP over Ethernet | ✅ | one at a time |
| HA ZBT-2 / Dongle-E | USB RCP | ✅ | one at a time |

**GL-S200** — runs OpenThread Border Router onboard with a web admin UI;
[GL.iNet's own HA docs](https://docs.gl-inet.com/iot/en/thread_board_router/gl-s200/work_with_home_assistant/)
**explicitly support Home Assistant Container** provided a Matter server runs in Docker — which
is exactly what CT 6001 already is. HA talks to it at `http://<ip>:8081`.

**SLZB-MR1** — spiritually the successor to our Cangji gateway: PoE, Ethernet, Wi-Fi, USB, web
UI, and it exposes **Thread on `:6638` and Zigbee on `:7638`** (independent EFR32MG21 + CC2652P7
radios, running at the same time). It could **replace the Tube gateway outright** and collapse
the XZG-vs-BLE-proxy tension in [#259](https://github.com/Chrison-Homelab/Homelab/issues/259)
into one box. Two catches: the OTBR stack runs on our host, and
[the reviewer warns plainly](https://smarthomescene.com/reviews/smlight-slzb-mr1-multi-radio-coordinator-setup-and-review/)
that *"the RCP protocol is not designed to be transferred over an IP network: it is a
timing-sensitive protocol."* Our Zigbee EZSP-over-TCP is fine; **Thread RCP is stricter** — do
not assume it inherits the Cangji box's reliability.

Useful either way, both avoiding the HAOS-only add-on:
- [`ownbee/hass-otbr-docker`](https://github.com/ownbee/hass-otbr-docker) — the HA OTBR add-on as a standalone container
- [`bnutzer/otbr-tcp`](https://github.com/bnutzer/docker-otbr-tcp) — OTBR built for **network-attached** Thread sticks

## The recommended path: ESP32-H2 RCP → OTBR container on CT 6001

Cheapest, and architecturally the best fit — it is the **official reference topology** (a Linux
host running `otbr-agent` with an ESP32-H2 RCP over UART), and it reuses what we own:

1. Flash an **ESP32-H2** with OpenThread RCP firmware
2. Plug it into the USB of the Proxmox node hosting **CT 6001**; pass the device through
   (bind-mount `/dev/ttyUSB*` + cgroup device rule)
3. Run **`openthread/border-router`** alongside `matter-server` in the same stack
4. Point HA's OTBR integration at `:8081`

Why this wins: real `otbr-agent`, **REST API on 8081 for free**, **Ethernet backhaul**, and it
lands on **IoT 1040 — the same L2 as the Matter server**, which kills the cross-VLAN mDNS risk
above. CT 6001 is already `network_mode: host` with IPv6 SLAAC, which is precisely what OTBR
wants.

The one property we give up is the no-USB-passthrough cleanliness we like about the Cangji box.
(Note: per [this stack's CLAUDE.md](../CLAUDE.md), CT 6001 stays **rootful** Podman under
ADR-0009 for exactly these host-net/BLE/Thread reasons — so a passed-through USB device fits
that exception rather than fighting it.)

### DIY alternatives, and their honest limits

**Single-chip ESP32-C6 border router** — officially supported by
[`esp-thread-br`](https://github.com/espressif/esp-thread-br) (Wi-Fi 6 + 802.15.4 on one die),
and there are [HA-targeted builds](https://github.com/l0cut15/esp32c6-thread-border-router).
Genuinely works, costs nothing, and we own two C6s. But:

- **ESP-IDF only — no ESPHome component.** It would sit *outside* the CT 6003 dashboard, unlike
  every other board in the fleet plan.
- **No OTBR REST API.** The firmware gives a serial CLI. Either provision the dataset once over
  CLI and let it advertise via mDNS `_meshcop._udp` (HA's Thread integration then finds it), or
  run an `otbr-proxy` shim — but the proxy needs the ESP **USB-tethered to a host**, which throws
  away the appliance property.
- **Wi-Fi is load-bearing and shares the RF path.** Wi-Fi and 802.15.4 can't receive
  simultaneously, and the BR advertises NAT64 over Wi-Fi — if Wi-Fi drops, **every Matter sensor
  goes offline.** A regression versus a PoE-backed wired box.
- Don't double-book a C6: the BLE-proxy build ([#251](https://github.com/Chrison-Homelab/Homelab/issues/251))
  is ESPHome, this is ESP-IDF, and the shared radio means one board shouldn't do both.

Espressif's own reference board (ESP32-S3 + ESP32-H2, ~NZ$35) has a proper web GUI, but
**Ethernet needs a separate sub-daughterboard** — the base board is Wi-Fi too.

### ❌ Dead end: the spare Raspberry Pi 1B

Considered and rejected. The Pi 1B is a BCM2835 / ARM1176JZF-S — **ARMv6** — and that fails twice:

1. **.NET requires ARMv7 minimum.** ARMv6 has never been supported (same reason the Pi Zero
   isn't). A C#/.NET daemon **cannot run on that board at all**.
2. **The official OTBR Docker images ship `amd64` / `arm64` / `arm/v7` only** — no armv6. And
   OpenThread's docs ask for a **Pi 3 or 4**.

More importantly, **writing our own daemon is the wrong shape of work.** OTBR already exposes the
REST API on `:8081` that HA expects — there is nothing to implement. And the REST layer is the
easy 5%; a border router is also NAT64, an **SRP server + Advertising Proxy** (bridges Thread
service registration into LAN mDNS — *mandatory* for Matter discovery), the Border Agent /
MeshCoP handshake, and RA / on-link / OMR prefix management. Reimplementing that is a multi-month
project that already exists, tested, in C++.

> If the appeal is owning and customising it: the reusable piece is a **C# client for the OTBR
> REST API**, not the daemon. That would slot into the existing `*Sharp` pattern and sit on top of
> a working border router instead of replacing one.

## Case study: IKEA TIMMERFLOTTE — do not buy it to fix slow sensors

IKEA **TIMMERFLOTTE** (`E2314`, 2×AAA, 5 dBm) is **Matter over Thread, not Zigbee** — IKEA
replaced most of their Zigbee sensor line. It will **never** join the Tube EFR32 coordinator.

**The disqualifier:** IKEA's Matter sensors report on a **firmware-defined interval (~5 min)** and
**the rate is not configurable** — for Thread-based Matter sensors the device owns its own
schedule, and there is no Matter equivalent of `zha_toolkit.conf_report`. That is the *same
failure mode* as our Tuya `EF00` sensors: the reporting mechanism simply isn't reachable. At
least Zigbee gives us a lever.

Also noted: a known HA Matter Server bug where the subscription hits a liveness timeout after
~30 min idle and doesn't resume without restarting the Matter server.

**Verdict:** fine to buy as a cheap, fully-local room thermometer with a display and as the
excuse to finally stand up Thread — IKEA Sylvia Park is 5 minutes away. **Not** a sensor upgrade,
and it will be *slower* than a properly configured SNZB-02D.

## Suggested order of work

1. **Confirm the Apple TV model** — decides whether a border router already exists here.
2. **Stand up the ESP32-H2 RCP + `openthread/border-router` on CT 6001.** Prove commissioning
   end-to-end on VLAN 1040 before buying any Thread device.
3. Only then consider the **GL-S200** as the durable appliance, if Thread earns its place.
4. Keep [sensor reporting](sensor-reporting.md) on **Zigbee** regardless — Thread does not help there.

## Related

- [`sensor-reporting.md`](sensor-reporting.md) — the actual fix for slow temp/humidity sensors
- [`../matter-server.lxc.yaml`](../matter-server.lxc.yaml) — the controller, its IPv6/mDNS requirements
- [`esp-fleet.md`](esp-fleet.md) — the ESP boards on hand and which chip does what
- [`devices/tube-zb-gw-efr32/`](devices/tube-zb-gw-efr32/) — the Zigbee coordinator this pattern is modelled on
- `~/marketplace-tools/` — `aliexpress.js` + the `thread_border_router` saved search
