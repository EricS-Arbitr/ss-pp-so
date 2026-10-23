# 02 — Network

All IPs, prefixes and interface assignments are owned by the blueprint and by
`host_vars`; this doc is the reconciled map. Re-derive from
`blueprints/ARBITR_PP_SecOnion-*.yml` after any topology change.

## Planes

| Plane | Range | Notes |
|---|---|---|
| Production (scenario) | `172.16.0.0/16`, `192.168.0.0/16`, `200.200.200.0/24`, `210.210.210.0/24`, `75.21.1.0/30` | What players and sensors see |
| Management (out-of-band) | `10.255.240.0/20`, gw/proxy `10.255.240.1` | Ansible path; hidden from scenario; mirror-excluded |
| Control (orchestration) | `control-10` = `10.10.0.0/16` | Ansible controller only (`ansible` = `10.10.10.10`) |

The management plane shares a data-plane NIC on the mirror-source routers, so it
is explicitly excluded from the mirror fabric (see below) and scrubbed from AD
DNS (see [03-active-directory.md](03-active-directory.md)).

## Host-bearing segments

| Segment | CIDR | Contents |
|---|---|---|
| Services | `172.16.2.0/24` | `pp-dc01`, `pp-dc02`, `pp-sql`, `pp-file`, `pp-proxy`, `pp-syslog`, `pp-corp-router` |
| Business Processing | `172.16.3.0/24` | `pp-bp-wkstn-1..8` |
| Engineering | `172.16.4.0/24` | `pp-eng-wkstn-1..8` |
| Legal | `172.16.5.0/24` | `pp-ls-wkstn-1..6` |
| InfoSec | `172.16.6.0/24` | `pp-is-wkstn-1..4` |
| DMZ | `172.16.8.0/24` | `pp-www` (.5), `pp-dmz-dns` (.4) |
| Security | `172.16.9.0/24` | SO grid (.30/.35/.40/.41/.42), `win-hunt-1..6` (.11–.16), `pp-sift` (.101), `pp-remnux` (.100), `pp-internal-router` (.1) |
| OT-Ctrl | `192.168.100.0/24` | `pp-dc03` (.5), `pp-dcs-ctrl` (.10), `pp-ctl-wks-01..04` (.101–.104) |
| Gas-Turbine | `192.168.95.0/24` | `pp-gasplant-plc` (.2), `pp-engws` (.5), `pp-gas-sim` (.100) |
| Gas-Turbine-Sim | `192.168.90.0/27` | `pp-gas-sim` (.8) |
| OT-Services | `192.168.90.96/27` | `pp-ot-hist` (.107), `pp-gasvibsens` (.101) |
| OT-DMZ | `192.168.90.128/27` | `pp-ot-malcolm` (.130) |
| Internet (sim) | `200.200.200.0/24` | `is-inet` (.2), `elgg` (.205) |
| red-net | `210.210.210.0/24` | `Red-1` (.2), `Activity-Emulation` (.70) |

## Point-to-point transit links (/30, over the `switch-*` segments)

| Link | CIDR | Endpoints |
|---|---|---|
| ISP ↔ external-fw | `75.21.1.0/30` | `pp-isp-router` .2 ↔ `pp-external-firewall` .1 |
| external-fw ↔ site-edge | `172.16.0.8/30` | `pp-external-firewall` .9 ↔ `site-edge-router` .10 |
| site-edge ↔ internal-fw | `172.16.0.16/30` | `site-edge-router` .17 ↔ `pp-internal-firewall` .18 |
| internal-fw ↔ internal-router | `172.16.0.24/30` | `pp-internal-firewall` .25 ↔ `pp-internal-router` .26 |
| internal-router ↔ corp-router | `172.16.0.40/30` | `pp-internal-router` .41 ↔ `pp-corp-router` .42 |
| internal-router ↔ ot-firewall | `172.16.0.48/30` | `pp-internal-router` .49 ↔ `pp-ot-firewall` .50 |
| ot-firewall ↔ ot-router | `192.168.200.200/30` | `pp-ot-firewall` .201 ↔ `pp-ot-router` .202 |

## Routing model

Verified against `host_vars`; see [`../../CLAUDE.md`](../../CLAUDE.md) for the
full rationale.

| Domain | Protocol | Notes |
|---|---|---|
| ISP edge | **eBGP** | `pp-isp-router` (AS 65002) ↔ `pp-external-firewall` (AS 65001). The **only** BGP session in the fabric. |
| Between the upstream firewalls | **OSPF area 0** | `pp-external-firewall` ↔ `pp-internal-firewall` exchange corp routes. `pp-external-firewall` redistributes OSPF into BGP (but **not** connected routes, keeping the management plane out of the ISP advertisement). |
| Corp core | **Static** | `pp-corp-router`, `pp-internal-router`, `site-edge-router` all carry `remove_vyos_bgp: true`. |
| OT boundary | **Static, default-deny** | `pp-ot-firewall` runs neither BGP nor OSPF. Corp reaches `192.168.0.0/16` via static route; OT reaches corp via default. |

The three pfSense firewalls run **FRR**. `pp-ot-router` is in
`[vyos_routes_only]` because its image (`RC-VyOS-Router`, OT variant) accepts
only raw static routes via `vyos_config`, not the base `vyos` role's interface
commands.

### Firewalls and routers

| Host | Type | Role in the fabric |
|---|---|---|
| `pp-external-firewall` | pfSense | perimeter; eBGP peer + OSPF; fronts the DMZ |
| `pp-internal-firewall` | pfSense | OSPF-only inner boundary |
| `pp-ot-firewall` | pfSense | Purdue L3.5 boundary, default-deny, static-only |
| `pp-isp-router` | VyOS | the ISP (AS 65002); excluded from syslog clients |
| `site-edge-router` | VyOS | static transit |
| `pp-corp-router` | VyOS | gateway for the five corp `/24`s |
| `pp-internal-router` | VyOS | gateway for `pp-security`; hub toward corp + OT |
| `pp-ot-router` | VyOS (`vyos_routes_only`) | head of the OT chain |

pfSense specifics (admin-user requirement, `php -r` escape hatches, FRR runtime
dir, NIC ordering) are in
[08-operations-and-troubleshooting.md](08-operations-and-troubleshooting.md) and
`roles/pfsense_firewall`.

## DNS

- AD-integrated zones for `voltgrid.com` on the domain controllers.
- The `dns` role creates zones/records but does **not** set forwarders; a
  `00-baseline.yml` play points both corporate DCs' forwarders at `is-inet`'s
  simulated public resolvers (`8.8.8.8`, `8.8.4.4`, `1.1.1.1`).
- `pp-dmz-dns` serves the DMZ; `global_dns` provides scenario name resolution
  (faa.gov, weather.gov, etc.) for the simulated Internet.
- Management-plane A records are scrubbed from AD DNS so `ping <host>` does not
  resolve to a `10.255.240.x` address.

## The GRE mirror fabric

Traffic is copied from router interfaces into `gretap` tunnels with `tc mirred`
and delivered to the sensors (and, for OT, to Malcolm). This is the backbone of
positional sensor coverage.

| Property | Value |
|---|---|
| Encapsulation | `gretap` (layer 2). A layer-3 `gre` tunnel yields `DLT_RAW`, which makes Zeek produce **zero** connection logs while Suricata keeps alerting — the classic "sensor looks up but sees nothing" failure. Both ends must be `gretap`. |
| Tunnel MTU | 1500 (raised above the kernel's 1462 `gretap` default); routers clamp TCP MSS to 1400 to keep mirrored frames within the routed underlay. |
| Mechanism | `tc` ingress + root `prio` qdiscs per source interface, `matchall action mirred egress mirror dev tun0`. On `pp-ot-router` the OT process interfaces also fan out to `tun1`, feeding Malcolm alongside the sensor. |
| Persistence | `/config/scripts/vyatta-postconfig-bootup.script`, re-applied at boot. |
| Management exclusion | `10.255.240.0/20` carried by two `action pass` filters at lower priority than the `mirred` catch-all, on each source interface. |

| Sensor | Mirror source | Coverage |
|---|---|---|
| `so-sensor-corp` | `pp-corp-router` eth0–eth4 | Services, Business, Engineering, Legal, InfoSec |
| `so-sensor-ot` | `pp-ot-router` eth1–eth4 | Gas-Turbine, Gas-Turbine-Sim, OT-Services, OT-Ctrl |
| `so-sensor-edge` | `pp-isp-router` eth1–eth2 | red-net and the simulated Internet |

Verify a mirror with the absolute path (`tc` is not on the `vyos` user's PATH):
`/sbin/tc filter show dev eth0 ingress` — expect one `mirred` filter and two
`action pass` filters per source interface.
