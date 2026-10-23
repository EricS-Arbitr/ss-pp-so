# 04 — Inventory and Roles

## How membership drives behaviour

The `hosts` INI inventory is the control surface for the whole build. Group
names are **load-bearing**: roles and plays address them by name
(`groups['so_manager']`, `hosts: sysmon`, `windows:linux:!unmanaged:!so_all`),
so renaming a group in `hosts` without renaming it in the roles breaks the build
silently. A host's capabilities and the telemetry it emits are decided entirely
by which groups it is in.

## Inventory groups (current expanded sizes)

Platform / OS groups:

| Group | Size | Meaning |
|---|---|---|
| `windows` | 43 | all Windows VMs (`:children` of the `win10`/`win11`/`winserver*` groups) |
| `linux` | 8 | `pp-proxy`, `pp-syslog`, `pp-www`, and the 5 SO grid nodes |
| `vyos` / `vyos_routes_only` | — | VyOS routers; `vyos_routes_only` = `pp-ot-router` (raw-static-only image) |
| `pfsense` | 3 | `pp-external-firewall`, `pp-internal-firewall`, `pp-ot-firewall` |

Role / posture groups:

| Group | Size | Meaning |
|---|---|---|
| `pdc` / `additional_dc` / `domain_controllers` | 1 / 2 / 3 | DC promotion roles |
| `members` | 39 | domain-joined non-DC hosts |
| `aue` | 30 | emulated-user workstations |
| `hunt` | 6 | analyst/hunt workstations (`win-hunt-1..6`, FLARE-imaged) |
| `sysmon` | 41 | hosts that run Sysmon (`[sysmon]` direct + children `aue` + `hunt`) |
| `forensics` | 2 | `pp-sift`, `pp-remnux` (label group; no play targets it) |
| `unmanaged` | — | hard-coded appliances no play touches: OT plant devices, `elgg`, and `pp-remnux` |
| `ae` | 6 | attack-emulation targets: `pp-dc01`, `pp-dc02`, `pp-dc03`, `pp-file`, `pp-sql`, `pp-dcs-ctrl` |
| `corporate_servers` | 3 | `pp-file`, `pp-sql`, `pp-syslog` |

Security Onion groups:

| Group | Size | Meaning |
|---|---|---|
| `so_manager` / `so_search` / `so_sensor` | 1 / 1 / 3 | grid node roles |
| `so_all` (`:children`) | 5 | every grid node |
| `so_mirror_routers` | — | the VyOS routers carrying a GRE mirror |
| `ansible_controller` | 1 | the mirror/controller host — deliberately **not** in `[linux]` |

Sizes are computed from the current inventory and move when hosts are
added/removed. The key derived numbers today: **72 hosts**, **43 Windows**,
**41 Sysmon** (exclusions `pp-dcs-ctrl`, `pp-dmz-dns`), **39 members**,
**6 AE targets**.

### Why `sysmon` is 41 not 43

`[windows]` is 43; `[sysmon]` is 43 minus the two Windows hosts intentionally
left out: `pp-dcs-ctrl` (process-critical OT equipment — event logs only, no
kernel driver) and `pp-dmz-dns` (outside the forest and the corporate telemetry
contract). Coverage is a *group*, not a host pattern, precisely so that "does
this host emit endpoint telemetry" is a recorded decision.

## `group_vars` hierarchy (lowest → highest precedence)

1. `group_vars/all/` — `main.yml` (proxy `inet_proxy_addr:port`,
   `syslog_server_ip`, shared tunables, vaulted secret *references*),
   `security_onion.yml` (the entire SO stack: pins, registry, subnets, airgap +
   Nexus source URLs), `vault.yml` (encrypted credentials).
2. `group_vars/<group>.yml` — `linux.yml`, `windows.yml`, `pfsense.yml`,
   `proxy.yml`, `voltgrid.yml`, `vyos_routes_only.yml`, `malcolm.yml`.
3. `host_vars/<host>.yml` — per-host IPs, interfaces, role-specific tuning.
4. Inline play vars in `playbooks/00-baseline.yml`.

No hardcoded IPs or credentials live in tasks or templates — `group_vars` /
`host_vars` are the single source of truth.

## Roles

Roles are pulled from `../range-development-ansible/roles/` first, then
overridden by the local `./roles/`. The tarball bundles ~29 base roles + 30
custom roles = ~50 total; the exact set is auto-discovered from every play (see
[05-build-and-deploy.md](05-build-and-deploy.md)).

### Custom / overlay roles

| Role | Purpose |
|---|---|
| `init` | Overlay. Waits for WinRM, then repairs and pins the default-gateway ARP entry (an impostor MAC answers ARP/ICMP for the gateway on some segments). |
| `common` | Overlay of the base baseline role. |
| `handlers` | Base handler set (includes Splunk handlers that no listener triggers here). |
| `group_assignment` | Dynamic group assignment that the static inventory cannot express on its own. |
| `additional_dc` | Promotes `pp-dc02`/`pp-dc03` into the `voltgrid.com` forest (proves the precondition before the expensive promotion). |
| `domain_member_retry` | Robust domain join with repair passes — the join path, not the member, is usually what fails on a cold build. |
| `dns` | AD-integrated zones + records; child-DC `DomainDnsZones` partition gate; record writes retry. |
| `dmz_public_dns` | `pp-dmz-dns`: file-backed public zone for the DMZ. |
| `global_dns` | Scenario public-name resolution (unbound container) for the simulated Internet. |
| `strip_apipa` | Removes `169.254.x.x` APIPA addresses Windows assigns during DHCP→static handoff. |
| `network_discovery` | Makes Windows hosts LAN-discoverable without the Public/Private prompt. |
| `disable_defender` | Overlay with PowerPlant-specific Defender tweaks. |
| `pfsense_firewall` | Drives pfSense 2.8.1 via `pfsensible.core` + `php -r` escape hatches (interfaces, gateways, default-gw pin, outbound-NAT, static routes, rules, FRR/BGP, syslog). |
| `vyos_mirror` | GRE `gretap` tunnels + `tc mirred` mirror rules feeding the sensors (and Malcolm). |
| `syslog_server` | `pp-syslog` as the central rsyslog collector (per-host files + agent fan-out). |
| `wordpress-pv` | Overlay: binds the WordPress container to `127.0.0.1:8080` so host nginx can vhost-route. |
| `voltgrid_site` | Populates the WordPress marketing site on `pp-www`. |
| `billing_site` | Voltgrid billing portal (Flask + gunicorn) on `pp-www`; Python deps from Nexus `ng_pypi`. |
| `cmms` | Voltgrid CMMS — SQL Server 2022 + maintenance DB on `pp-sql`. |
| `cmms_client` | CMMS client config on the engineering workstations. |
| `docker_engine` | Installs Docker (consumed by Malcolm, WordPress, etc.); honours the Nexus insecure-registry / proxy config. |
| `so_apt_mirror` | nginx on the controller serving the SO source snapshot, airgap content, ETOPEN rules and the skopeo-built container registry. |
| `so_base` | Prereqs on every grid node before `so-setup`. |
| `so_manager` | Renders the answer file, seeds the registry from the mirror, runs `so-setup`, verifies the outcome (not the exit code). |
| `so_search` | Search-node install + grid join (`SEARCHNODE`). |
| `so_sensor` | Sensor install + grid join, GRE decap, Zeek/Suricata (note the lifetime-invariant guard before the `end_host` probe). |
| `so_fleet_integrations` | Host-scoped Fleet policies + integrations (nginx, squid, pfSense, VyOS) driven directly against the Fleet API. |
| `elastic_agent` | Installs/enrols the Elastic Agent on every `[windows]`+`[linux]` endpoint from controller-staged installers. |
| `malcolm` / `malcolm_host_prep` | Deploys the CISA Malcolm OT appliance (26.08.0, images pulled from Nexus and retagged to `ghcr.io`); host prep for the Expanded image. |

### Adding a role

Don't edit the role list in `build_tarball.sh` — it auto-discovers from plays.
Add a role by referencing it in a play (`roles:`, `import_role` or
`include_role`), and it is bundled on the next build.
