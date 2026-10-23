# 08 — Operations and Troubleshooting

## Credentials

Stored in `group_vars/all/vault.yml` (ansible-vault). The values below are the
current range **defaults** — rotate them before any deployment that is not a
lab.

| System | Access | Username | Password |
|---|---|---|---|
| Security Onion SOC | `https://172.16.9.30` | `admin@voltgrid.com` | `Simspace1!Simspace1!` |
| Domain admin | `voltgrid.com` | `simspace` | `Simspace1!Simspace1!` |
| Domain users | `voltgrid.com` | emulated identities | `Simspace1!Simspace1!` |
| Windows servers | RDP / WinRM | `simspace` | `Simspace1!Simspace1!` |
| Windows 10/11 workstations | RDP / WinRM | `xadmin` | `ConfigingInTheNameOf1!` |
| Linux hosts | SSH | `simspace` | `simspace1` |
| pfSense firewalls | SSH / web | `admin` | `simspace1` |
| VyOS routers | SSH | `vyos` | `Simspace1!` |
| SO node join secret | — | — | `Simspace1!Simspace1!` |
| Malcolm UI | `https://192.168.90.130` | `analyst` | (vaulted) |

Workstation and server local accounts **differ** (`xadmin` vs `simspace`) — the
workstation images ship a different account. Using the wrong one does not fail
fast; it retries to timeout and looks like an unbooted host. The hunt
workstations autologin, so a player at one has a desktop without credentials.

### Rotation

```bash
ansible-vault edit group_vars/all/vault.yml --vault-password-file /home/simspace/.vault_pass
```

Tasks that embed a credential in a shell command carry `no_log` — Ansible echoes
`cmd` verbatim on failure and `deploy.sh` tees that to
`/var/log/playbook_run.log`, so without it a failure would write a Domain Admin
password to disk in clear. Credentials passed as **module parameters** need
nothing: `microsoft.ad` / `ansible.windows` mark those `no_log` in their own
argspec.

## Known behaviours (look like faults, are not)

- **SOC health-check failures fetching detection content.** SOC tries GitHub
  every five minutes for rule updates, AI summaries and playbooks. The range has
  no egress, so these fail continuously. Detection on ingested data is
  unaffected.
- **`SA-cim_vladiator` export warning.** Cosmetic; the app ships that way.
- **Emulated-user noise.** 32 users browsing/opening/moving files is
  intentional. Detections that only work on a quiet network are not detections.
- **`INIT_GW_REPAIRED_AND_PINNED` during deploy.** The platform-level gateway
  ARP fault being caught and corrected before it breaks routing. Seeing it is
  good — the guard is working.
- **A first-attempt host failure.** Usually ordering, not fault — the host
  depends on state another reaches asynchronously. Attempt 3 is the
  confirmation (see [05-build-and-deploy.md](05-build-and-deploy.md)).

## Common pitfalls

1. **VyOS image bakes self-loop default routes.** FRR refuses a default whose
   next-hop resolves to a local interface, so `show ip route` has no
   `S>* 0.0.0.0/0` even though host_vars declares one. Fix with
   `extra_static_routes_remove` per host.
2. **`next_hop` typo: own /30 IP instead of the peer's.** The inline
   `# pp-X-router` comment next to each next_hop exists because of exactly this.
3. **Windows DDNS registers both adapters** → `ping pp-dc01` resolves to a
   management IP half the time. The baseline strip play fixes it;
   `Resolve-DnsName -DnsOnly` is the reliable check.
4. **pfSense `/cf/conf/config.xml` permission denied as non-admin.**
   `ansible_user: admin` is mandatory on pfSense — `simspace` can't copy onto
   the root-owned config file.
5. **pfSense FRR runtime dir missing on boot** (`/var/run/frr/`); the
   `pfsense_firewall` role pre-creates it before `service frr onestart`.
6. **`gretap` vs `gre`.** Zeek silent, Suricata alerting = layer-3 tunnel. Both
   ends `gretap`.
7. **pfSense NIC ordering.** On the 2.8.1 image mgmt is on the **first** NIC
   (`vmx0`); a `host_vars` referencing `vmx0` as a data-plane interface is
   stale.
8. **The `dns` role sets no forwarders** — non-`voltgrid.com` names don't
   resolve until the baseline forwarders play runs.

The authoritative, dated list of overlay workarounds is
[`../../UPSTREAM_FIXES.md`](../../UPSTREAM_FIXES.md) — read it before changing
network or AD plumbing; half the time the problem is already diagnosed there.

## Re-deploying and resetting

```bash
# Re-run one phase during development
ansible-playbook playbooks/40-manager.yml

# Targeted recovery (e.g. a single DC + the Fleet gate on the manager)
BOOT_DELAY=0 ansible-playbook site.yml --limit pp-dc01,so-manager

# Full rebuild on an already-up range
BOOT_DELAY=0 ./deploy.sh
```

A `--tags reset` scope (data-only vs also-config) is a design knob — confirm
intent before wiring destructive resets.

## Re-pinning the blueprint to a new tarball

The deploy chain is driven by the blueprint, not git. After building and
pushing a new `ab_pp.tgz`:

1. Build, commit and push the tarball (see
   [05-build-and-deploy.md](05-build-and-deploy.md)); note the **full** commit
   SHA.
2. Edit `blueprints/ARBITR_PP_SecOnion-*.yml` → the
   `download-range-vars-script` `content:` → replace the SHA in the
   `raw.githubusercontent.com/EricS-Arbitr/ss-pp-so/<sha>/ab_pp.tgz` URL.
3. Confirm: `grep -o "ss-pp-so/[0-9a-f]*/ab_pp.tgz" blueprints/…yml`.
4. A fresh range fetches that exact tarball via `wget … -e https_proxy=
   10.255.240.1:3128` (curl does **not** work through this proxy on the
   controller; wget does).

Blueprints are not bundled in the tarball, so this change needs no rebuild — but
it only takes effect on a **fresh** range built from the edited blueprint, not
on a running one (re-pull the tarball by hand to update a live controller).

## Operating gotchas worth internalizing

- **Nexus is reachable only via the mgmt proxy, never directly**, and presents
  an internal-CA cert. Any new fetcher you add must route through
  `10.255.240.1:3128` and trust/skip-verify `nexus.dev.ng.simspace.lan`.
- **The `/home/simspace/.vault_pass` file does not persist across spin-ups** —
  recreate it on every fresh controller or `deploy.sh` fails closed.
- **Verify the committed tarball, not the working copy** (`git show HEAD:ab_pp.tgz`)
  — a stale or partially-staged tarball has silently shipped before.
- **`docs/` and `blueprints/` are not deployed** — editing them never changes a
  running range.
