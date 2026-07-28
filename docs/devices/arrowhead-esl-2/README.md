# Arrowhead ESL-2 (ELITE-S) alarm panel

Home security panel by **Arrowhead Alarm Products** (NZ). **Discontinued / effectively
abandonware.** Goal: integrate it into Home Assistant **locally** (zone status,
arm/disarm, outputs, system health) **without** buying the paid RS232 board and
**without** the cloud — by tapping the panel's keypad bus with an **ESP32**.

## What we have
- **RS232-BD V2 protocol manual** ([`RS232-BD-V2-protocol.pdf`](RS232-BD-V2-protocol.pdf))
  from Arrowhead support — fully documents the RS232 ASCII protocol the board
  emits/accepts. Transcribed to [`rs232-protocol.md`](rs232-protocol.md).
- **ESL-2 Installation & Programming Guide V9** ([`ESL-2-Install-and-Program-Manual.pdf`](ESL-2-Install-and-Program-Manual.pdf))
  — the official AAP manual for *our* panel. 136 pp, text-searchable. Keypad port is
  documented on **p34**. 21 MB, and already optimally compressed (re-running Ghostscript
  `/ebook` makes it *bigger*) — don't bother trying to shrink it.
- **ELITE-S Installation & Programming Guide v905** ([`Elite-S-Manual-v905.pdf`](Elite-S-Manual-v905.pdf))
  — same panel family, same **V9** firmware generation, and **the better document of the two**:
  120 pp but ~5× the extractable text, a full connection diagram (p5), and the programming
  reference the ESL-2 guide compresses. Reach for this one first.
- **Hardware on hand:** several **ESP32**, one **Raspberry Pi 1 rev B**.
- Panel + keypad bus (physical access at home).

### Confirmed from the manuals (2026-07-28)

The keypad-bus wiring assumed throughout this document is **correct**:

> "The terminals marked **POS, NEG, CLOCK, & DATA** make up the communications port which the
> keypads and other intelligent field devices use to talk to the controller."
> — ESL-2 guide, p34

and the optional fifth wire is confirmed too — ELITE-S p18: the **`LIN`** terminal is a
*listen-in* line carrying call-progress audio to the keypad (gated on program option
`P175E 6E`). It is **not** part of the data bus, so a read-only tap ignores it.

Program-mode entry, from ELITE-S p27 — note this is a **panel-wide** mode, not per-keypad:

```
<PROGRAM> - <Installer Code> - <ENTER>        → the PROGRAM light FLASHES
Default installer code (address P25E1E) = 000000
```

Two corrections to what was previously recorded here from web snippets:

- **There is no `IPGM` display.** The string appears **zero times** in either manual; entry is
  signalled by the *Program LED flashing*. (LCD keypads show menu headings such as
  `INSTALLER:USERS` instead — ELITE-S p29.)
- **Exit is not a two-key `<PROG> <ENTER>`.** On an LCD keypad you *repeatedly* press
  `<PROG>` until the display reads `<ENTER> TO EXIT`, then press `<ENTER>`. Local (per-keypad)
  Program Mode is a different thing again, exited by holding `<PROGRAM>` for two seconds.

One operational constraint worth knowing before bench work: **"Program mode access is
inhibited if any part of the system is Armed"** (ELITE-S p27).

> **Still missing:** the RS232-BD *keypad* instruction sheet on `manuals.plus` — that host
> returns **HTTP 403** to non-browser clients, so it needs a manual save from a browser. Low
> value now: ELITE-S p18 already documents the 5-wire keypad port it was wanted for, and the
> board's RS232 side is fully covered by [`rs232-protocol.md`](rs232-protocol.md).

## How the official path works (the board we're avoiding)
The **RS232-BD** is a small board that:
1. Connects to the ELITE-S **keypad bus** — 4 wires: **POS, NEG** (power) + **CLK, DAT** (clocked synchronous serial).
2. **Impersonates a keypad** at an unused address (DIP switches → KP#1–8; must be an *unused* address or the bus conflicts).
3. Translates the keypad bus ↔ **RS232 ASCII, 9600 8N1** (DB9: Tx→pin2, Rx→pin3, GND→pin5).

So the board's *entire paid value is the bus↔RS232 translation.* The manual documents
the **RS232 side** (every event + command) — see [`rs232-protocol.md`](rs232-protocol.md).

## Keypad bus — already decoded (Crow Runner == Arrowhead AAP)
**Huge shortcut:** Crow Runner and Arrowhead AAP are the same hardware family, sharing this
exact `POS/NEG/CLK/DAT` keypad bus — and the community has **already reverse-engineered it**
(see Prior art). From `sivann/crowalarm` (Pi) + `MadDoct/ESP-CrowAlarmInterface` (ESP):

- **Signal levels: CLK/DAT are 5 V logic** (only `POS` is ~12 V — the *power* rail). So
  level-shifting is easy: **resistor divider** (read-only) or a **bi-directional logic-level
  converter** (needed for control). Tie **NEG → MCU GND**.
- **Sampling:** read **DAT on the CLK falling edge**. **DAT is active-low** (`0 = high`).
- **Framing: HDLC-like** — frames delimited by the flag byte **`10000001`**, ~**72-bit** frames.
  **Zones are a bitmap** in the frame (in the Runner's layout, bits 24–31 = zones 1–8, a
  `0` = active). ELITE-S/ESL-2 has **16 zones**, so *our field map will differ* and needs
  confirming on our panel — but the mechanics (levels, edge, flag, framing) should carry.

So this is **"adapt + map fields", not "RE from scratch."**

## The plan
**Phase 0 — capture our panel's bus.** Divider on CLK/DAT → an ESP32 (or the Pi 1B running
`sivann/crowalarm`); find the `10000001` flags, dump frames, and **map the field layout**
using the RS232-BD semantics as ground truth: open zone 1 → see which bit flips; arm → find
the arm-state bits; etc. (16 zones + areas A/B + system flags = more fields than the 8-zone
Runner example.)

**Phase 1 — passive sniff → HA (read-only, zero risk).** Decode frames → publish zones /
arm state / mains-battery-tamper / outputs to **MQTT (broker CT 6000)** → HA.

**Phase 2 — control (bidirectional).** MadDoct already does this two ways: **write on the
bus** (needs the level converter), or **relay-simulated keyswitches**. Either gives
arm/disarm/outputs. Do after Phase 1 is solid.

**Firmware:** port `MadDoct/ESP-CrowAlarmInterface` (ESP8266 → **ESP32**) or adapt
`sivann/crowalarm` (**Pi 1 rev B**, read-only) — both already target hardware you own.
Surface via **MQTT** (MadDoct ships HA MQTT-Alarm-Control-Panel configs). Later a proper
SmartHome stack member.

> **Trivial fallback** if the bus map proves stubborn on the ELITE-S: the RS232-BD board
> *does* speak the fully-documented ASCII protocol ([`rs232-protocol.md`](rs232-protocol.md))
> — ESP32 + MAX3232 + a genuine board. Avoiding that purchase is the whole point, so bus-tap first.

## Prior art (evaluated 2026-07-17)

**★ The shortcut — direct CLK/DAT keypad-bus taps (same bus as ours):**
- **[MadDoct/ESP-CrowAlarmInterface](https://github.com/MadDoct/ESP-CrowAlarmInterface)** — **ESP8266** on CLK/DAT. 5 V→3.3 V via level converter (bi-dir) or divider. **HDLC-like** decode. **Bidirectional** (read + arm/disarm via bus *or* relay-simulated keyswitches). **MQTT → HA** with ready configs. → **primary port target for our ESP32.**
- **[sivann/crowalarm](https://github.com/sivann/crowalarm)** — **Raspberry Pi Model B** (== our Pi 1 rev B) reads active zones off CLK/DAT in Python. Documents the framing (flag `10000001`, CLK-falling, DAT active-low, zone bitmap). → **read-only reference + handy capture tool.**

**Transport-mismatch (not our path, but confirm ESL-2 is in the family / reuse semantics):**
- **[febalci/ha_pycrowipmodule](https://github.com/febalci/ha_pycrowipmodule)** — TCP to an **IP Module / "ESL-2 APP POD"** (needs that hardware + special AAP firmware). Explicitly lists **"AAP Elite ESL-2"** → confirms ESL-2 ∈ AAP/Crow family.
- **[thanoskas/arrowhead_alarm](https://github.com/thanoskas/arrowhead_alarm)** — TCP "Serial-over-IP", **ECi-series only** now (v1.x did ESX Elite-SX, unmaintained). Not ESL-2, but a reference HA entity/state model.
- **[ankohanse/hass-elite-cloud](https://github.com/ankohanse/hass-elite-cloud)** — ESL-2 via **Elite Cloud** (cloud — avoid).

**Manuals / community:** [RS232-BD (manuals.plus)](https://manuals.plus/arrowhead-alarm/rs232-bd-elite-s-keypad-manual) (403s to non-browsers) · [ESL-2 install/programming (ManualsLib)](https://www.manualslib.com/manual/2040574/Arrowhead-Alarm-Products-Esl-2.html) — now held locally, see [What we have](#what-we-have) · [HA forum — EliteControl NZ](https://community.home-assistant.io/t/question-about-integrating-elitecontrol-alarm-system-into-home-assistant-nz-company/402663) · [HA forum — Crow ESP8266 interface](https://community.home-assistant.io/t/crow-runner-alarm-interface-using-an-esp8266-and-home-assistant/629939) · [Geekzone — Arrowhead HomeKit](https://www.geekzone.co.nz/forums.asp?forumid=73&topicid=306147).

## Next steps
1. **Read `MadDoct/ESP-CrowAlarmInterface` + `sivann/crowalarm` source** end-to-end; lift the CLK/DAT decode + wiring.
2. **Wire a read-only tap** — resistor divider on CLK/DAT (5 V→3.3 V), NEG→GND — to an ESP32 (or the Pi 1B running `crowalarm` as-is for a first capture).
3. **Capture our ELITE-S bus** — dump `10000001`-flagged frames while triggering known events, and **map the ESL-2 field layout** (16 zones, areas A/B, system flags) against [`rs232-protocol.md`](rs232-protocol.md).
4. **Port MadDoct → ESP32**, read-only first → **MQTT (CT 6000) → HA**.
5. **Phase 2:** enable control (bus write via level converter, or relay-sim keyswitches).
6. Ship as a SmartHome stack member; consider contributing the ELITE-S field map back upstream.

## ⚠️ Safety
It's a **live security panel**. Bus-tap **read-only first**; never disrupt the panel's own
monitoring/dialler. The panel keeps functioning independently of anything we attach.
