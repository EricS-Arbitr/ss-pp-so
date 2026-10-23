# 07 — Verification

Verification happens in two places: **build time** (gates in
`build_tarball.sh` that refuse to ship a bad archive) and **runtime** (health
checks after a deploy that assert *outcomes*, not exit codes).

## Build-time gates

Covered in [05-build-and-deploy.md](05-build-and-deploy.md#build-time-gates). In
short, `build_tarball.sh` runs `verify_vars.py`, `verify_shell_args.py`,
`verify_so_inventory.py`, `verify_dup_keys.py`, `verify_handlers.py`,
`verify_task_keywords.py` and `verify_defend_filters.py`, and aborts the build
if any fails. Expected `verify_vars.py` warning count against the stage is **4**;
a different number means something changed.

Run `verify_vars.py` against the **stage**, not the repo root — the stage
includes base roles copied from `../range-development-ansible/`, so vars like
`nat` and `splunk_admin_password` appear only there. `build_tarball.sh` already
runs it against the stage; run it the same way by hand.

## Runtime verification

Two scripts on the deployed controller:

```bash
sudo ./verify_so.sh -v          # Security Onion grid (prints its own totals)
sudo ./verify_deployment.sh     # range-wide, read-only health check
```

For `verify_so.sh` **the number that matters is `Fail: 0`.**

Then confirm the thing that most often fails silently — that sensors are
*seeing traffic*, not merely running:

```bash
sudo ansible so-sensor-corp,so-sensor-edge,so-sensor-ot -b -m shell \
  -a 'timeout 10 tcpdump -i tun0 -c 20 2>&1 | tail -3'
```

A sensor with Suricata alerting but Zeek producing zero connection logs means
the tunnel is carrying the wrong frame type — both ends must be `gretap`
(see [06-security-onion-and-telemetry.md](06-security-onion-and-telemetry.md)).

Verify a mirror itself with the absolute path (the `vyos` user has no `tc` on
PATH, and a bare invocation returns "command not found" that a counting grep
cannot tell from "no filters"):

```
/sbin/tc filter show dev eth0 ingress
```

Expect one `mirred` filter and two `action pass` filters per source interface.

## What "healthy" looks like

| Check | Expected |
|---|---|
| `so-status` on every SO node | all services running |
| Elastic cluster | green, search node present |
| `tcpdump -i tun0` on each sensor | packets flowing |
| Zeek `conn.log` | connection records, not zero |
| Fleet | every endpoint enrolled |
| Domain membership | 39 hosts joined to `voltgrid.com` (42 counting the 3 DCs) |
| Local registry (airgap) | ≥ 22 repos before `so-setup` proceeds |

## Three-layer post-deploy check

### L1 — routing convergence

On each VyOS router:
```
show ip bgp summary       # every neighbor Established (only pp-isp-router ↔ pp-external-firewall)
show ip route bgp
show ip route 0.0.0.0/0   # default in FIB
```
On each pfSense (use `sh -c` under tcsh):
```sh
vtysh -c "show ip bgp summary"
vtysh -c "show ip route"
netstat -rn -f inet | awk '$1=="default"||$1~/^172|^192|^10|^75/'
```
"The config has the line" is not "the kernel installed the route" — image-baked
junk routes, equal-cost statics and self-loop next-hops all make FRR drop routes
from the FIB. Confirm the FIB, not the config.

### L2 — pfSense config integrity

```sh
xmllint --xpath 'string(//gateways/defaultgw4)' /cf/conf/config.xml; echo
xmllint --xpath 'string(//nat/outbound/mode)' /cf/conf/config.xml; echo
pfctl -vsr | grep -c 'label "USER_RULE'
```

### L3 — end-to-end from a corp workstation

```powershell
Test-NetConnection -ComputerName 172.16.8.5 -Port 80   # DMZ reachable
tracert -d 192.168.95.2                                 # OT reachable
Resolve-DnsName pp-dc01 -DnsOnly                         # AD DNS only, not LLMNR
ping pp-dc01                                             # resolves to 172.16.2.7, not a mgmt IP
```

## Verification philosophy

Gates assert what the next step actually needs, and they must block only on
**self-clearing** conditions — a latched flag makes an `until:` unpassable, and
`failed_when: false` does not rescue an exhausted `until` (use `ignore_errors`).
A gate that passed "last time" proves nothing when the thing it guards is
timing-dependent. This is why `so_manager` verifies repos-present and
services-up rather than the `so-setup` exit code, and why the `dns` play gates
on the `DomainDnsZones` partition rather than assuming it exists.
