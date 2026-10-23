# 03 — Active Directory

## Forest

`voltgrid.com`, single forest, single domain, **three** domain controllers.

| DC | Address | Placement | Inventory group |
|---|---|---|---|
| `pp-dc01` | `172.16.2.7` | Primary, Services | `pdc` |
| `pp-dc02` | `172.16.2.8` | Additional, Services | `additional_dc` |
| `pp-dc03` | `192.168.100.5` | Additional, **inside the OT enclave** | `additional_dc` |

`pp-dc03` is the deliberate one: the OT control workstations authenticate
against a DC on their own side of `pp-ot-firewall`, so routine OT identity
traffic does not cross the boundary. When Kerberos/NTLM to a corporate DC *does*
appear from OT, that is itself a detection signal.

Promotion order matters: `pp-dc01` is promoted first (`pdc`), then `pp-dc02` and
`pp-dc03` join as additional DCs via the `additional_dc` role (there is no
sibling role for this in the base repo).

## Membership

- **39 domain members** (`[members]`) + 3 DCs = 42 domain-joined hosts.
  Members = the 30 corporate/OT workstations, the 6 hunt workstations, and
  `pp-file`, `pp-sql`, `pp-dcs-ctrl`.
- `pp-dmz-dns` sits in the DMZ, **outside** the forest.
- The OT plant devices and the forensics VMs are not domain-joined.

Counts are derived from the inventory; re-derive after inventory changes (see
[04-inventory-and-roles.md](04-inventory-and-roles.md)).

## Emulated users

32 emulated users with real identities (`charity.bowen`, `ahmed.ortega`,
`david.miller`, …) run on the corporate workstations (`[aue]`, 30 hosts). They
log in, browse, open documents and move files continuously — this is the
background noise detection logic must survive. Hunt workstations are **not** in
`[aue]`: analyst activity is never mixed with synthetic user activity on the
boxes analysts work from.

## DNS and the management plane (the recurring hazard)

Every host is dual-homed, with a management NIC on `10.255.240.0/20`. Windows
registers **both** adapters into AD DNS by default, so `ping pp-dc01` can resolve
to a management IP half the time. Two mitigations are wired in:

1. A baseline play strips the management interface from AD DNS registration and
   scrubs `10.255.240.x` A records. (The base `common` role's own attempt at
   this has known bugs — a typo'd adapter name and the wrong cmdlet parameter —
   so the overlay play is the real fix; see `../../UPSTREAM_FIXES.md`.)
2. DCs force-register and prove their locator SRV records (`nltest /dsregdns`)
   before any member join runs, because a fresh build sometimes starts joins
   before Netlogon has registered them.

`Resolve-DnsName <host> -DnsOnly` is the gold-standard check — `ping` and
`nslookup` both have LLMNR/NetBIOS footguns on the shared management L2.

## DNS forwarders

The `dns` role does not set forwarders, so anything outside `voltgrid.com` is
unresolvable on domain workstations until a baseline play points both corporate
DCs at `is-inet`'s simulated public resolvers. This is deliberate: it keeps the
range self-contained while still giving players name resolution for scenario
domains.

## Child-DC / DNS-partition convergence note

On a cold build a freshly promoted additional DC can answer ADWS minutes before
its `DomainDnsZones` application partition is enlisted, and DNS record writes
against it fail until it is. The `dns` play gates on the partition being present
(probing it, creating the built-in partitions with `dnscmd` if absent,
restarting the DNS service, re-probing) and record writes retry rather than
failing the host. This is the same class of cold-build convergence issue tracked
across the Blackstone/airfield ranges; keep the gate and the record retry if you
refactor the `dns` play.
