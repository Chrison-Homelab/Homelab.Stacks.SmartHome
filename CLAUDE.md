# CLAUDE.md — Homelab.Stacks.SmartHome

Guidance for Claude Code working in this repo.

## What this is

A **stack submodule** of [`Chrison-Homelab/Homelab`](https://github.com/Chrison-Homelab/Homelab),
mounted there at `stacks/SmartHome` (meta-repo model, ADR-0008). This repo holds only the stack's
own shapes and assets; the **converge/validate engine lives in the superproject** and is the only
thing that applies them.

> **Read the superproject's [`CLAUDE.md`](https://github.com/Chrison-Homelab/Homelab/blob/main/CLAUDE.md) first.**
> It carries the rules that apply here and are *not* repeated in this file: the PR-only git
> workflow + merge strategy (ADR-0010), the worktree rule for parallel sessions, the shared
> external-account guardrails (**add-only** — never touch Cloudflare/UniFi resources we didn't
> create), and `secrets.env` / Bitwarden Secrets Manager handling.

## This stack

The home-automation / **IoT support layer** — the infrastructure that feeds Home Assistant,
rather than HA itself.

- **CTID block:** `6000–6099` (enforced by `stack.yaml`'s `ctidRange`)
- **Network:** IoT **VLAN 1040**
- **Exposure:** **INTERNAL-ONLY.** MQTT is never exposed publicly and there is no tunnel ingress
  for any member. Clients on other VLANs reach the broker via inter-VLAN routing.

| ID | Member | Notes |
|---|---|---|
| 6000 | `mqtt` (Mosquitto) | Native community-scripts install; the broker everything else uses |
| 6001 | `matter-server` | Host-net + Thread/BLE |
| 6002 | `aircast` | Host-net for mDNS/RTP |
| 6003 | `esphome` | ESPHome dashboard |
| **4100** | `leapmotor-mate` | ⚠️ **Adopted in-place, OUT-OF-BLOCK** — live at 4100, kept to avoid a redeploy. **Now stopped and tagged `retired`** — Mate runs on the podman host (`podman-host/quadlets/leapmotor-mate.container`), so this shape is dead weight pending the CT's deletion |
| **2000** | `homeassistant` | ⚠️ **Adopted — `manage: describe-only`** VM; predates this stack and is never written by converge |

**The two out-of-block members are deliberate, documented exceptions.** Do not "fix" their IDs —
renumbering means recreating the guest.

VM 2000 carries **`spec.manage: describe-only`** (#325), so "treat it as read-only" is now enforced
by the engine rather than by this sentence: plan reports it as `DESCRIBE-ONLY` instead of perpetual
drift, `--apply` reports `SKIPPED`, and **naming it in `--only` does not override that**. Before the
marker existed, an unscoped `converge --apply` on this stack proposed a `SetConfig` against an
adopted HAOS install, and the only thing stopping it was remembering to scope the run.

## Working here

Converge runs **from the superproject**, pointed at this directory — never from inside this repo:

```bash
# in the superproject
dotnet run --project Infrastructure/engine -- validate stacks/SmartHome
dotnet run --project Infrastructure/engine -- converge stacks/SmartHome          # dry run
dotnet run --project Infrastructure/engine -- converge stacks/SmartHome --apply
```

Shapes validate against the superproject's `Infrastructure/schema/shape.schema.json`. This repo
also runs an opt-in `validate.yml` that calls the superproject's reusable `_validate-shapes.yml`;
it needs the `SCHEMA_RO_PAT` Actions secret to be in scope for this repo.

## Gotchas specific to this stack

- **Podman migration is in flight.** ADR-0009 migrates the Docker-in-LXC members to rootless
  Podman + quadlets; the platform landed in Phase 0 (superproject #284). Before authoring any
  quadlet file, read `docs/plans/284-podman-platform.md` in the superproject — in particular that
  a literal `%` must be written `%%` in `Exec=`, and that boot survival requires
  `[Install] WantedBy=default.target`.
- **The host-net members (`matter-server` 6001, `aircast` 6002) are deliberately NOT going
  rootless.** They need `network_mode: host` for mDNS/RTP and Thread/BLE plus
  `apparmor=unconfined`; rootless buys ~nothing on an IoT-edge box. They stay rootful Podman.
- **MQTT credentials live in Bitwarden** (items `mqtt · <client>`), never in a shape. Broker
  passwords are set with `mosquitto_passwd`; discovery prefix is `homeassistant`.
- **Device tokens / bind keys** for the flashed Xiaomi/BTHome sensors are in a Bitwarden note, not
  in this repo.
