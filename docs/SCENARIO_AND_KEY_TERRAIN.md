# Voltgrid Power — Scenario, Cyber Key Terrain & Vulnerability Reference

Mission framing, cyber key-terrain tiering and the catalogue of deliberate
weaknesses for the `voltgrid.com` power-utility range.

Audience: event coordinators, scenario designers and red/blue leads. This
document complements [`RANGE_GUIDE.md`](RANGE_GUIDE.md) (topology, telemetry,
deploy) and [`RANGE_OVERVIEW.md`](RANGE_OVERVIEW.md) with the training-scenario
material: who the range *is*, what matters most on it, and where it is
deliberately weak.

> **Two reading conventions.** The **Scenario** is fiction — a backstory to hang
> an exercise on. The **Cyber Key Terrain** and **Intentional Vulnerability
> Catalog** are grounded in the real build: every host named exists in
> [`hosts`](../hosts) and the `RANGE_GUIDE` inventory, and every weakness listed
> is actually present in the Ansible roles or the platform images. Where a
> specific CVE or patch level matters to an exercise, confirm it against the
> deployed SimSpace image build — the catalogue below lists the deliberate,
> *structural* weaknesses, not an asserted CVE inventory.

---

## 1. Scenario

### The organization

**Voltgrid Power Company** is a regional electric utility operating a
combined-cycle, gas-turbine generating station. It sells power to municipal and
commercial customers and is subject to the usual reliability and data-protection
obligations of a utility.

Its network splits the way a real utility's does:

- A **corporate enterprise** (`voltgrid.com`) — Services, Business Processing,
  Engineering, Legal and InfoSec — where staff do business on domain-joined
  Windows workstations, authenticated by two domain controllers.
- A **public face** in the DMZ — a WordPress marketing site and a customer
  **billing portal** (`billing.voltgrid.com`) backed by a SQL database holding
  customer and billing records.
- An **OT enclave** behind its own firewall — the Distributed Control System
  (DCS) control room, the gas-turbine and its simulation, a process historian
  and a vibration sensor. A third domain controller (`pp-dc03`) lives *inside*
  the enclave so plant operators authenticate without crossing the boundary.

### The mission

Two things must not fail: **generation must stay up and stable**, and the
**control process must stay trustworthy**. Secondarily, **customer billing data
must stay confidential and intact**. A Security Operations Center on the
security segment — Security Onion, six hunt analysts, forensics and
malware-analysis benches, and Malcolm for OT deep-packet analysis — exists to
detect and contain intrusions before they reach either.

### A representative exercise storyline

One of many an operator can run on this terrain — written here to show how the
pieces connect, not as the only path:

1. **Entry.** A corporate user is phished through the simulated Internet mail
   path (`is-inet`). Code runs on a corporate workstation. Windows Defender is
   disabled fleet-wide, so execution is quiet to AV — but Sysmon and Security
   Onion still see it.
2. **Lateral movement.** The attacker reuses a shared local-administrator
   credential (`xadmin` / `simspace`) to move through the Services segment and
   lands on the attack-emulation target set — `pp-dc01`, `pp-dc02`, `pp-sql`,
   `pp-file`.
3. **Two divergent objectives.**
   - *Data track:* abuse the billing portal's seeded IDOR and reflected-XSS
     weaknesses to read other customers' bills and harvest sessions.
   - *Process track:* pivot across the OT boundary (`pp-ot-firewall`) toward
     `pp-dc03` and the DCS control station (`pp-dcs-ctrl`), then issue
     unauthenticated Modbus writes to the turbine PLC. Malcolm's ICS parsers
     surface the process-layer activity that ordinary network logs cannot.
4. **Defence.** Because sensor coverage is positional, the blue team must
   correlate the intrusion across `so-sensor-corp`, `so-sensor-edge` and
   `so-sensor-ot` — identity traffic crossing the OT boundary is itself the
   signal.

The same terrain supports purely-web scenarios, insider scenarios, and
OT-first scenarios; the key-terrain and vulnerability tables below are what an
author draws on to build them.

---

## 2. Cyber Key Terrain (CKT)

Cyber Key Terrain is the infrastructure, systems and data whose compromise most
hurts the mission. It is tiered by the confidentiality/integrity/availability
(CIA) guarantee each asset must hold:

| Tier | Guarantee required | Compromise impact |
|---|---|---|
| **Tier 1** | 100% confidentiality **and** integrity **and** availability | Grave; halts the core mission (generation, the control process, identity, or the detection mission itself). |
| **Tier 2** | 100% confidentiality **and** integrity | Serious; interrupts or degrades operations. |
| **Tier 3** | 100% integrity | Degraded performance; recoverable. |

### Infrastructure CKT — routers and firewalls

| Device | Tier | CIA | Effect of compromise |
|---|---|---|---|
| `pp-ot-firewall` | 1 | A, I | The OT boundary. Loss or misconfiguration opens the path from corporate to the physical process; a malware-bearing but "up" firewall is an integrity failure. |
| `pp-ot-router` | 1 | A | Gateways the plant segments **and** feeds the OT mirror to `so-sensor-ot` and Malcolm. Loss severs process reachability and blinds OT monitoring at once. |
| `pp-external-firewall` | 2 | A, I | Edge NAT/BGP boundary and the Internet-facing control point; integrity failure if it forwards hostile traffic as trusted. |
| `pp-corp-router`, `pp-internal-router`, `site-edge-router` | 2 | A | Corporate reachability and the corporate/edge mirror feeds. Compromise degrades both connectivity and the detection picture. |
| `pp-isp-router` | 2 | A | Edge transit and the `so-sensor-edge` mirror of the attack path. |

### Servers and services CKT

| Device | Tier | CIA | Effect of compromise |
|---|---|---|---|
| `pp-dc01`, `pp-dc02`, `pp-dc03` | 1 | C, I, A | Active Directory authentication. `pp-dc03` sits inside the OT enclave; its compromise bridges identity straight into the control network. |
| `pp-dcs-ctrl` | 1 | A, I | The DCS control station — issues process commands. Compromise has a physical consequence for the turbine. |
| `pp-gasplant-plc`, `pp-gas-sim`, `pp-ot-hist`, `pp-gasvibsens` | 1 | A, I | The physical process and its instrumentation. Unmanaged, unauthenticated ICS endpoints; loss of integrity misrepresents or destabilises generation. |
| `so-manager`, `so-search`, `so-sensor-corp/-ot/-edge` | 1 | A, I | The detection mission. Loss blinds the defenders; falsified telemetry is the most damaging integrity failure the SOC can suffer. |
| `pp-sql` | 2 | C, I, A | Customer and billing database; the billing portal's backend. |
| `pp-www` | 2 | C, I, A | Public WordPress + billing portal (`billing.voltgrid.com`); seeded web weaknesses make it a likely first web target. |
| `win-hunt-1..6` | 2 | C, I | Analyst positions. An attacker here sees everything the SOC sees and can steer the investigation. |
| `pp-sift`, `pp-remnux` | 2 | C, I | Forensic and malware-analysis benches — hold investigation work product; `pp-remnux` shares layer 2 with the SO grid (detonation-adjacency risk). |
| `pp-file` | 2 | C, A | Corporate file server; internally sensitive documents. |
| `pp-proxy`, `pp-syslog` | 2 | A | Egress control and the local log archive — a realistic attacker target for log tampering. |
| `pp-dmz-dns`, `pp-www` (marketing) | 3 | A | Public-facing name resolution and content; recoverable. |
| Corporate workstations (`[aue]`) | 3 | I | Footholds for entry and lateral movement; individually recoverable. |

---

## 3. Intentional Vulnerability Catalog

These weaknesses are present **by design** for training. They are not defects to
fix — several are self-documented in source — and each pairs a weakness with the
telemetry that makes it detectable, because the point of the range is to practise
seeing them.

> This repository is public, and these weaknesses are intentional range content
> (the billing app documents its own seeded flaws in source; the `RANGE_GUIDE`
> already describes the disabled AV, the unmanaged OT and the shared accounts).
> Still: **rotate all default credentials before any deployment that is not a
> lab.** CWE and ATT&CK identifiers below are the weakness *class*; confirm any
> version-specific CVE against the deployed image.

| # | Component / host | Weakness | Class | Lives in | Detection opportunity |
|---|---|---|---|---|---|
| 1 | Billing portal, `pp-www` | **IDOR** on `/bill/<id>` — bills fetched by primary key with no ownership check; IDs are enumerable | CWE-639; ATT&CK T1190 / T1213 | role `billing_site` (`app.py`) | `logs-nginx.access-*` shows one session walking sequential `/bill/` IDs |
| 2 | Billing portal, `pp-www` | **Reflected XSS** — the `error` query param is rendered client-side via `innerHTML` on `/login` | CWE-79; ATT&CK T1059.007 | `billing_site` (`login.html`) | script-like payloads in the nginx access-log query string |
| 3 | Billing portal, `pp-www` | **Hardcoded session secret** — Flask `SECRET_KEY` falls back to `voltgrid-dev-secret-change-me`, enabling session forgery | CWE-798 / CWE-321 | `billing_site` (`app.py`) | anomalous authenticated sessions without a preceding login |
| 4 | All Windows domain members | **Windows Defender disabled** fleet-wide via GPO (antispyware, real-time, behaviour, on-access, cloud-delivered all off) | CWE-1188; ATT&CK T1562.001 | role `disable_defender` | the GPO object is enumerable; Sysmon (41/43) still fires, so the *absence* of Defender events is itself a baseline signal |
| 5 | Plant devices (`pp-gasplant-plc`, `pp-gas-sim`, `pp-ot-hist`, `pp-gasvibsens`) | **Unauthenticated ICS protocols** — Modbus, OPC-UA, S7, DNP3, EtherNet/IP with no auth or encryption | CWE-306; ATT&CK T0836 / T0855 | protocol-inherent; unmanaged images | Malcolm ICS parsers on `pp-ot-malcolm` — unexpected write function codes against the PLC |
| 6 | Plant devices (as above) | **Unmanaged / unpatched** — no config management; legacy images, as real plant equipment behaves | CWE-1104 | by design (`[unmanaged]`) | `so-sensor-ot` network baseline; new flows to/from plant devices |
| 7 | `pp-dcs-ctrl` | **No endpoint instrumentation** (no Sysmon, deliberate OT posture) — process trees and Sysmon network/DNS events are absent | visibility gap (not an injected flaw) | `[sysmon]` exclusion in `hosts` | reconstruct from Windows event logs + `so-sensor-ot` only |
| 8 | `pp-www` (WordPress) | **WordPress attack surface** — plugin/theme/xmlrpc exposure typical of a public CMS | ATT&CK T1190 | role `wordpress-pv` | `logs-nginx.access-*` |
| 9 | `elgg` | **Legacy social-platform** web app, unmanaged — a known-vulnerable application class | ATT&CK T1190 | `[unmanaged]` scenario host | `so-sensor-edge` / `so-sensor-corp` |
| 10 | Windows & Linux fleet | **Shared local-admin / default credentials** — servers `simspace`, workstations `xadmin`, Linux `simspace`, pfSense `admin`, VyOS `vyos` | CWE-798; ATT&CK T1078 / T1550 | `group_vars` / vault | SO authentication datasets; one admin credential used across many hosts |
| 11 | `is-inet` | **Email ingress** — inbound mail to corporate users is the designed phishing path | ATT&CK T1566 | scenario mail host | `so-sensor-corp`; mail flow to corporate segments |
| 12 | `pp-remnux` | **Detonation adjacency** — shares layer 2 with the SO grid and hunt hosts, so a worm-like sample reaches them without traversing a router | operational risk (documented) | `[forensics]` / `[unmanaged]` | contain to static analysis; see `RANGE_GUIDE` "Adjacency" |

### Using the catalogue

- A **web-exploitation** exercise draws on #1–3, #8, #9.
- An **OT / process-integrity** exercise draws on #4–7.
- An **identity / lateral-movement** exercise draws on #4, #10, #11 and the
  attack-emulation target set.

Each row names the role or inventory group that produces the weakness, so an
author can enable, disable or harden it deliberately rather than by accident.
