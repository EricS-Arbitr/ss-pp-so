# 01 — Architecture

## What this range is

Voltgrid Power is a 72-host simulated electric utility built as a **defensive
cyber range** on the SimSpace NG platform. It exists to train and assess SOC
analysts against a network that behaves like a real IT/OT estate: a real Active
Directory forest, a real DMZ, a segmented OT enclave running gas-turbine plant
equipment, continuous emulated-user noise, and a distributed Security Onion grid
watching from three vantage points.

The whole range is **Ansible from a pinned tarball**. It stands up, re-deploys
and resets idempotently so an exercise can be run, torn down and rebuilt
clean.

`ss-pp-so` is a **range-specific overlay** on top of the customer's shared
platform repo (`../range-development-ansible/`). The overlay ships inventory,
`host_vars`, `group_vars`, custom roles and range playbooks, then bundles the
base roles it uses plus its own overlays into `ab_pp.tgz`.

## Design decisions that shape everything downstream

These four properties are load-bearing — most behaviours only make sense
against them.

1. **Sensor coverage is positional.** Each Security Onion sensor receives a
   layer-2 mirror from exactly one router. An event is visible only to the
   sensor whose router carries it; there is no aggregation tap. Correlating an
   intrusion across segments means correlating across sensors, and *which*
   sensor carries an event is itself a location signal. See
   [06-security-onion-and-telemetry.md](06-security-onion-and-telemetry.md).

2. **The OT enclave is segmented at layer 3.** `pp-ot-firewall` is a
   default-deny boundary; OT reaches corporate over static routes only. A third
   domain controller (`pp-dc03`) lives *inside* the enclave so OT hosts
   authenticate locally and routine identity traffic never crosses the
   boundary — which makes it a signal when it does.

3. **Endpoint telemetry is deliberately non-uniform.** 41 of 43 Windows hosts
   run Sysmon; the exclusions are policy, not oversight, and are expressed as
   inventory-group membership so "does this host emit endpoint telemetry" is a
   decision someone recorded. See
   [04-inventory-and-roles.md](04-inventory-and-roles.md).

4. **The network is not quiet.** 32 emulated users browse, open documents and
   move files continuously across the corporate segments. Detection logic that
   assumes a clean baseline will not behave as it does in a lab.

Platform- and customer-level decisions (control plane on `10.255.240.0/20`,
pfSense firewalls, the OSPF+eBGP+static routing model, role-sourcing policy) are
recorded in [`../../CLAUDE.md`](../../CLAUDE.md) and are not repeated here.

## System architecture

```
                 is-inet (200.200.200.2) — simulated Internet / elgg
                                 │
                 pp-isp-router (VyOS, AS 65002)
                                 │  eBGP  75.21.1.0/30
                 pp-external-firewall (pfSense, AS 65001) ── DMZ 172.16.8.0/24 (pp-www, pp-dmz-dns)
                                 │  static 172.16.0.8/30
                 site-edge-router (VyOS, static)
                                 │  static 172.16.0.16/30  (OSPF adj to internal-fw)
                 pp-internal-firewall (pfSense, OSPF-only)
                                 │  static 172.16.0.24/30
                 pp-internal-router (VyOS, static) ── pp-security 172.16.9.0/24 (SO grid, hunt, forensics)
                     │                        │
         172.16.0.40/30               172.16.0.48/30
                     │                        │
         pp-corp-router (VyOS)        pp-ot-firewall (pfSense, static, default-deny)
         corp /24s:                            │  static 192.168.200.200/30
           Services   172.16.2.0/24   pp-ot-router (VyOS, CLI-only: vyos_routes_only)
           Business   172.16.3.0/24   OT subnets:
           Engineering172.16.4.0/24     Gas-Turbine      192.168.95.0/24
           Legal      172.16.5.0/24     Gas-Turbine-Sim  192.168.90.0/27
           InfoSec    172.16.6.0/24     OT-Services      192.168.90.96/27
                                        OT-DMZ           192.168.90.128/27 (pp-ot-malcolm)
                                        OT-Ctrl          192.168.100.0/24  (pp-dc03, DCS, 4 wkstns)
```

Every host is additionally dual-homed onto the **management plane**
(`10.255.240.0/20`, gateway/proxy `10.255.240.1`). Ansible connects over
management; the data plane never routes to it. The Ansible controller itself
sits on a separate control subnet (`control-10`, `10.10.0.0/16`,
`ansible 10.10.10.10`) and is **not** in `[linux]`.

Full addressing and routing detail is in [02-network.md](02-network.md).

## The three bespoke subsystems

- **Corporate IT** — `voltgrid.com` AD forest, five corporate departments,
  DMZ, continuous emulated users. See
  [03-active-directory.md](03-active-directory.md).
- **OT / gas-turbine plant** — a Purdue-style enclave behind `pp-ot-firewall`:
  a DCS control station and control workstations (`OT-Ctrl`), the plant PLC,
  historian, vibration sensor and engineering workstation (`Gas-Turbine`,
  `Gas-Turbine-Sim`, `OT-Services`). The plant devices are **unmanaged** —
  they arrive pre-configured from their images and Ansible never touches them,
  exactly as plant equipment behaves under a corporate config-management
  regime.
- **Security / SOC** — the Security Onion grid, six FLARE-imaged hunt
  workstations, and two uninstrumented forensics VMs, all on `pp-security`.
  OT traffic additionally feeds a CISA Malcolm appliance for ICS-protocol deep
  packet analysis.

## Telemetry / data flows

| Source | Path into the SIEM | Dataset |
|---|---|---|
| Network (corp / OT / edge) | layer-2 `gretap` mirrors → the three sensors → Zeek + Suricata | Zeek `conn`/`dns`/`http`/`files`, Suricata alerts |
| Windows + Linux endpoints | Elastic Agent → SO Fleet | endpoint (Defend/osquery/system) + Sysmon on the Windows subset |
| nginx (`pp-www`) | agent tails vhost logs | `logs-nginx.*` |
| Squid (`pp-proxy`) | Squid emits UDP → local agent | `logs-squid.log-*` |
| pfSense ×3 / VyOS ×n | devices → `pp-syslog` (rsyslog) → local agent | `logs-pfsense.*`, `logs-vyos-*` |
| OT process traffic | second `gretap` tunnel → `pp-ot-malcolm` | Malcolm (Arkime/Zeek ICS parsers) |

Everything lands in one platform, parsed into its own dataset, so correlating a
proxy request with the process that made it and the firewall rule that allowed
it is one investigation, not three.

## Artifact sourcing (air-gapped)

The range has no general internet egress. Install artifacts come from the
in-platform **Nexus** (`nexus.dev.ng.simspace.lan`), reached through the
management proxy: the Security Onion source + container registry, Malcolm's
source + images, the Ansible Galaxy collections, and the billing-portal Python
packages. **apt is the exception** — base packages and `docker-ce` install from
the Ubuntu archive and `download.docker.com` through the proxy, because the
Nexus apt repo serves several Ubuntu releases from one index and no pin can
safely prefer the right one. Detail in
[05-build-and-deploy.md](05-build-and-deploy.md).
