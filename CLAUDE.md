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

## Tracking

Issues for this stack live in **this repo's tracker**. The superproject keeps cross-repo epics and
anything that changes the engine.

- **File stack-local work here.** Shapes, members, device wiring, stack docs.
- **File engine work in the superproject.** A story that needs converge to learn something new —
  for example [Homelab#383](https://github.com/Chrison-Homelab/Homelab/issues/383) (reconcile
  multi-NIC `networks[]`) — goes there and gets linked, because the engine is not in this repo.
- **Labels mirror the superproject taxonomy.** Apply one category label at creation time, per the
  superproject's [`docs/agents/issue-and-pr-style.md`](https://github.com/Chrison-Homelab/Homelab/blob/main/docs/agents/issue-and-pr-style.md).
  That file also defines the issue shape (`### Problem` / `### Outcome` / `### Acceptance criteria`).

### Home Assistant milestone

The **Home Assistant** milestone tracks moving HA off the adopted HAOS VM 2000 onto a
converge-managed container LXC (CT 6005). The cross-repo epic is
[Homelab#250](https://github.com/Chrison-Homelab/Homelab/issues/250) and every story is attached to
it as a sub-issue, so the epic shows progress across both repos.

Two things to know before picking up any of it:

- **The critical path is the engine, not the shape.** `spec.networks[]` is descriptive for every
  guest kind today (`VmConverger.cs:56`, `CtConfigReconciler.cs:11`, and the community-scripts
  create path emits one `var_vlan`), so the dual-homed LXC the migration calls for cannot be
  converged until Homelab#383 lands. Building it by hand would recreate the drift this migration
  exists to remove.
- **Do not size CT 6005 from the numbers in Homelab#250.** They were measured at 2 cores / 2 GB.
  VM 2000 was raised to 4 cores / 4 GB on 2026-08-09 after HA pegged both cores and timed out
  bootstrap, cancelling 14 integrations (Homelab#382). `homeassistant.vm.yaml` still records the
  old figures and is drifted from live.

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

## Build & release

This repo has its own **Fallout 10.4** pipeline (`build/Build.cs`). Fallout 10.4 is public on
nuget.org, so there is no package-feed PAT here — unlike the superproject, which is still pinned
to the abandoned `2026.1.0-preview` edge line on GitHub Packages.

```bash
./build.sh                              # ValidateShapes (default)
./build.sh Bundle                       # + dist/ artifact + MANIFEST.md
./build.sh Release --dry-run            # + version resolution, no publish
./build.sh Bundle --skip ValidateShapes # macOS/Windows — the portable validator is linux-x64 only
```

Three things to know before touching it:

- **`./build.sh` in this repo does not deploy.** It validates and packages. Converge needs the
  engine and cluster credentials, which stay in the superproject (see below). A `Deploy` target is
  expected later; the seam is the `Engine(...)` helper, which already runs the portable binary.
- **The PR label decides the version.** `build/Build.cs` reads the labels on every PR merged since
  the last tag: `breaking-change` → major, `enhancement` → minor, else patch. A label is no longer
  just changelog dressing — mislabel a `ctid` change as `enhancement` and the release understates
  that guests get recreated.
- **Two `gh` calls need two different tokens.** Downloading the validator reads the *private
  superproject* (`SCHEMA_RO_PAT`); resolving PR labels and creating the release read *this* repo
  (the ambient Actions token). The schema token is scoped to that one process rather than exported
  across the run — don't "simplify" it into a single global `GH_TOKEN`.

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
