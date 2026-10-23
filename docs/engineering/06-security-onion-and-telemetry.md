# 06 — Security Onion and Telemetry

Security Onion is the range's **only** SIEM; every log source reports to it.
Malcolm adds ICS-protocol deep-packet analysis for the OT enclave but is not a
SIEM.

## The grid

Distributed Security Onion 2.4 on `pp-security` (`172.16.9.0/24`):

| Node | Address | Role |
|---|---|---|
| `so-manager` | `172.16.9.30` | Manager, SOC web UI (`https://172.16.9.30`), Fleet |
| `so-search` | `172.16.9.35` | Search node (`SEARCHNODE`) |
| `so-sensor-corp` | `172.16.9.40` | Corporate traffic |
| `so-sensor-ot` | `172.16.9.41` | OT traffic |
| `so-sensor-edge` | `172.16.9.42` | Attack path + simulated Internet |

Pins live in `group_vars/all/security_onion.yml`:

| Var | Value | Meaning |
|---|---|---|
| `so_git_ref` | `55af7eb…` | SO 2.4/main HEAD snapshotted 2026-07-22 |
| `so_registry_image_version` | `2.4.211` | SO image set |
| `so_registry_registry_version` | `3.0.0` | registry container |
| `so_registry_elasticsearch_version` | `9.0.8` | Elasticsearch |
| `so_airgap_mode` | `true` | no internet during setup |
| `so_registry_min_repos` | `22` | setup is gated on this many repos present in the local registry |

The SOC web UI is configured for **IP access** (`BASIC` advertisement), so
there is no hostname to resolve — reach it from a hunt workstation at
`https://172.16.9.30`.

## Airgap install: the registry seed

`so-setup` runs in airgap mode and pulls container images from a **local
registry**, never ghcr/dockerhub. The flow:

1. `so_apt_mirror` (on the controller) serves the SO source snapshot, the
   airgap content (`so_airgap_artifact_base`), the ETOPEN ruleset, and the
   container-registry artifacts.
2. The registry images are staged as `registry.tar` / `registry_image.tar`
   **before** `so-setup` runs — `so-setup` deletes the docker proxy drop-in
   mid-run, so anything not already local is unreachable.
3. `so_manager` seeds the registry, renders the answer file, runs `so-setup`,
   and verifies the **outcome** (repos present, services up) rather than the
   exit code. Setup is gated on `so_registry_min_repos` (22) being present.

With `so_registry_source` pointing at `nexus-docker.dev.ng.simspace.lan/
security-onion-solutions`, the controller pulls the registry contents from
Nexus; with `so_registry_artifact_url` set to a raw base it downloads a prebuilt
tarball; otherwise it builds the registry locally with skopeo. No in-play system
needs internet access.

## Positional sensor coverage

Each sensor is fed by exactly one router's `gretap` mirror (see
[02-network.md](02-network.md) for the `tc`/`gretap` mechanics). There is no
aggregation tap, so:

- An event on `so-sensor-ot` and absent from `so-sensor-corp` is traffic that
  did **not** cross the OT boundary.
- Cross-segment activity must be correlated across sensors, not read from one
  stream.
- The management plane is excluded from every mirror (`action pass` for
  `10.255.240.0/20` ahead of the `mirred` catch-all), so players never see the
  orchestration plane.

The single most common sensor failure is a `gre` (layer-3) tunnel where a
`gretap` (layer-2) tunnel is required: Suricata keeps alerting while Zeek
produces **zero** `conn.log` records. Both tunnel ends must be `gretap`.

## Endpoint telemetry

`elastic_agent` enrols every `[windows]` (43) and `[linux]` (8) host into SO's
Fleet from controller-staged installers. **Sysmon** runs on the `[sysmon]`
subset (41 Windows hosts; `pp-dcs-ctrl` and `pp-dmz-dns` excluded — see
[04-inventory-and-roles.md](04-inventory-and-roles.md)).

`pp-dcs-ctrl` is the instructive exclusion: it carries an Elastic Agent (reads
the Windows event logs Windows already writes) but **no** Sysmon (no kernel
driver on process-critical OT equipment). On that host you have logons, account
use and service installs plus network evidence from `so-sensor-ot`, but no
process tree — activity must be reconstructed from those two sources. That is
how conservative OT programmes actually instrument.

## Application log sources

Four non-endpoint sources reach SO as **parsed datasets**, each on a dedicated
Fleet agent policy created by `so_fleet_integrations`:

| Source | Host | Dataset | How |
|---|---|---|---|
| nginx | `pp-www` | `logs-nginx.access-*` / `error-*` | agent tails both vhost logs |
| Squid | `pp-proxy` | `logs-squid.log-*` | Squid emits UDP to the local agent |
| pfSense ×3 | via `pp-syslog` | `logs-pfsense.log-*` | rsyslog fans the raw datagram to a local listener |
| VyOS ×n | via `pp-syslog` | `logs-vyos-*` | agent reads the collector's per-host files |

`pp-www` fronts both sites (`billing.voltgrid.com` → gunicorn,
everything else → WordPress) through one nginx, so one access log covers both.
Firewalls/routers can't run an agent, so `pp-syslog` relays for them — the
pfSense copy is relayed **byte for byte** (re-rendering the syslog header breaks
the pfSense ingest pipeline's parsing). VyOS has no vendor integration, so its
bodies stay as text with `log.file.path` carrying the router name.

### Fleet-integration rules worth knowing

- Derive the input id, never pattern-match it: the key is
  `<policy_template>-<input_type>`, and a package's policy template is not
  always named after the package (Squid's is `log`, so its input is `log-udp`,
  not `squid-udp`).
- A dedicated policy contains only what you put in it, so moving a host off
  `endpoints-initial` strips its endpoint telemetry unless the base
  integrations are mirrored too (`mirror_base: true`).
- Renaming an integration does not replace the old one (`…_check` matches by
  name), so add the old name to `so_fleet_retired_integrations` in the same
  change.
- `so_fleet_integrations` drives `elastic_fleet_integration_create/_update`
  directly rather than `so-elastic-fleet-integration-policy-load`, which walks
  only hardcoded directories and would never read one we add.

## Malcolm — OT deep-packet analysis

`pp-ot-malcolm` (`192.168.90.130`, OT-DMZ, mgmt `10.255.240.220`) is a CISA
Malcolm appliance dedicated to the OT enclave. SO sees the same OT traffic;
Malcolm adds the ICS parsers (OPC-UA, Modbus, S7, DNP3, EtherNet/IP) that make
process traffic legible.

| Property | Value |
|---|---|
| Version | Malcolm **26.08.0** (OpenSearch, Arkime, Zeek, Suricata, Logstash) under Docker |
| Image source | pulled from `nexus-docker.dev.ng.simspace.lan/idaholab/malcolm` and **retagged locally to their `ghcr.io` names** — the compose stays `ghcr.io` because `control.py`'s `keystore_op` greps the compose for `image: ghcr.io/idaholab/malcolm/<svc>`; rewriting the refs breaks keystore creation and the stack never starts |
| Source | `ng_raw` tarball, unpacked into the install dir (gates the git clone) |
| Startup | the systemd unit runs `control.py` with `--profile malcolm` so the opensearch keystore is created |
| Capture | live on `otmir0` (Zeek/Suricata/netsniff-ng read the decapsulated mirror) |
| Feed | a second `gretap` tunnel (`tun1`) from `pp-ot-router` carrying the process segments only |
| Web UI | `https://192.168.90.130`, basic auth `analyst` |

`pp-ot-router` mirrors only the process segments to Malcolm (PLC on
Gas-Turbine, simulation on Gas-Turbine-Sim, historian/vibration on OT-Services).
The Windows OT-Ctrl domain is **not** sent — the ICS parsers add nothing to AD
traffic and `so-sensor-ot` already covers it. The tunnel is receive-only.
