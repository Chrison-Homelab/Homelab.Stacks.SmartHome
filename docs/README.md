# SmartHome docs

Documentation for the smart-home layer — the **purchased end-devices** (bulbs,
plugs, sensors, appliances, cast targets, the car) that the SmartHome *services*
(MQTT broker, Matter Server, AirCast, Home Assistant) talk to.

> The **services/infrastructure** are documented in the stack
> [`../README.md`](../README.md). This folder is about the **things** — what we
> own, where they are, and how they integrate. It's the durable, version-controlled
> replacement for the lapsed Confluence page, and will be surfaced via Docusaurus
> (issue #252).

| Doc | Contents |
|-----|----------|
| [`devices.md`](devices.md) | Purchased smart-home device inventory |
| [`sensor-reporting.md`](sensor-reporting.md) | Why the temp/humidity sensors report slowly, and how to fix it — the Tuya `EF00` root cause, `zha-toolkit` reporting config, the battery ceiling, ZigbeeTLc, and what to buy |
| [`thread-and-matter.md`](thread-and-matter.md) | Thread/Matter state of play — we have a controller but **no border router**; the onboard-OTBR vs RCP decision, and why Matter sensors don't fix reporting speed |
| [`esp-fleet.md`](esp-fleet.md) | Bare ESP boards on hand for DIY builds, and which chip suits which job |
