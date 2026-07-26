# podman-host (CT 6004) — the SmartHome stack's rootless Podman + quadlet host

The stack's **container host**, not a per-app box. Services arrive as extra quadlets in
[`quadlets/`](quadlets/) rather than as new one-app Docker CTs. Established by ADR-0009
(rootless Podman + quadlets, systemd + git as the control plane) and Phase 1 / #285.

- **CT 6004** — in-block for this stack (6000–6099), unlike the CT 4100 it replaces
- **VLAN 1040** (IoT), internal-only, no tunnel ingress
- **Rootless user** `podman`, subuid window `10000:50000` (fits the LXC's own uid map)
- Read the superproject's `docs/plans/284-podman-platform.md` **before editing a quadlet** —
  its eight gotchas are all load-bearing (notably: escape a literal `%` as `%%`, and every
  container quadlet needs `[Install] WantedBy=default.target` or it won't start at boot)

## Current members

| Quadlet | Service | Notes |
|---|---|---|
| `leapmotor-mate.container` | Leapmotor Mate | Migrated from CT 4100; publishes to the MQTT broker (CT 6000) |

## Why the Mate cutover needs a short downtime

Not a limitation of Podman — a constraint of the upstream service:

**Leapmotor allows ~one active session per account.** Two Mate instances logged into the
same dedicated account will keep kicking each other out, so the old and new containers
**cannot both run logged-in at once**. The host and unit are built side-by-side safely
(image pulled, unit valid), but the actual handover is exclusive.

Second constraint: **22 MB of live state** lives in CT 4100's `/opt/leapmotor/data` — the
SQLite telemetry history, the uploaded mTLS client certs, *and* the auto-generated
`MATE_SECRET_KEY`. That key encrypts the stored account credentials, so the data directory
must move as one piece. (`MATE_SECRET_KEY` is blank in CT 4100's `.env`, which is exactly
why it lives inside `data/` — do **not** set it in the quadlet, or the migrated database
becomes undecryptable.)

MQTT settings are configured in Mate's own UI and stored in that SQLite DB, so they come
across with the data — no re-wiring of the broker or Home Assistant is needed.

## Cutover runbook

Assumes `converge stacks/SmartHome --apply` has already created CT 6004 and rendered the
quadlet (the unit will be up but unconfigured until the data lands).

```bash
# 0. from the superproject, provision the host + quadlet
dotnet run --project Infrastructure/engine -- converge stacks/SmartHome --apply

# 1. stop the OLD Mate — frees the Leapmotor account session
ssh root@hpe-01 'pct exec 4100 -- docker stop leapmotor-mate'

# 2. stop the NEW unit before seeding its data dir
ssh root@<node-of-6004> 'pct exec 6004 -- bash -lc "cd /; U=\$(id -u podman);
  runuser -u podman -- env XDG_RUNTIME_DIR=/run/user/\$U \
    DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/\$U/bus \
    systemctl --user stop leapmotor-mate.service"'

# 3. copy the state across (via the node, then into 6004), preserving ownership
ssh root@hpe-01 'pct exec 4100 -- tar -C /opt/leapmotor -cf - data certs' > /tmp/mate.tar
ssh root@<node-of-6004> 'pct exec 6004 -- mkdir -p /home/podman/leapmotor'
ssh root@<node-of-6004> 'pct exec 6004 -- tar -C /home/podman/leapmotor -xf -' < /tmp/mate.tar
ssh root@<node-of-6004> 'pct exec 6004 -- chown -R podman:podman /home/podman/leapmotor'
rm -f /tmp/mate.tar        # contains mTLS certs + the secret key — do not keep it around

# 4. start the new unit and watch it come up
#    (same runuser incantation as step 2, with `start` then `is-active`)

# 5. verify — in this order
#    a. unit active + container healthy
#    b. Mate web UI on http://<6004-ip>:4000 shows the EXISTING history (not a setup wizard)
#    c. MQTT: broker CT 6000 sees leapmotor/<VIN>/… topics again
#    d. Home Assistant entities live (they are discovery-based, so they re-populate)

# 6. leave CT 4100 STOPPED but NOT destroyed — deliberate rollback path.
ssh root@hpe-01 'pct stop 4100'
```

**Rollback** at any point: `pct start 4100 && pct exec 4100 -- docker start leapmotor-mate`,
then stop the new unit. The old CT keeps its own copy of the data, untouched by the copy
above — nothing is moved, only duplicated.

CT 4100 is left stopped rather than destroyed on purpose; deleting it is a manual decision
once the new host has proven itself over a few days.
