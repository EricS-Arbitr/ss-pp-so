# ss-pp-so — Engineering Documentation Pack

Engineering reference for the **Voltgrid Power** (`voltgrid.com`) Security Onion
cyber range, as built by the `ss-pp-so` Ansible overlay. This pack is for
engineers who **build, extend, operate and debug** the range — not for players
or instructors.

For the operator/instructor view see [`../RANGE_GUIDE.md`](../RANGE_GUIDE.md);
for the one-page capability summary see
[`../RANGE_OVERVIEW.md`](../RANGE_OVERVIEW.md). This pack goes deeper into design
rationale, the build/verify pipeline, the inventory and role model, and the
failure modes you will actually hit.

## Contents

| Doc | Covers |
|---|---|
| [01-architecture.md](01-architecture.md) | Purpose, design decisions, system architecture, data/telemetry flows, segmentation model |
| [02-network.md](02-network.md) | Addressing plan, transit links, routing model, firewalls, DNS, the GRE mirror fabric |
| [03-active-directory.md](03-active-directory.md) | Forest, three DCs, the in-OT DC, members, emulated users, DNS/mgmt hazards |
| [04-inventory-and-roles.md](04-inventory-and-roles.md) | Inventory groups and their meaning, host→group mapping, custom + base roles, group_vars hierarchy |
| [05-build-and-deploy.md](05-build-and-deploy.md) | Tarball build, build-time gates, blueprint fetch+deploy, the 3-attempt retry model, phase order, vault, Nexus artifact sourcing |
| [06-security-onion-and-telemetry.md](06-security-onion-and-telemetry.md) | SO grid, sensor placement, registry seeding, Fleet integrations, endpoint telemetry, Malcolm OT DPI |
| [07-verification.md](07-verification.md) | Build-time verifiers, runtime health checks, what "healthy" looks like |
| [08-operations-and-troubleshooting.md](08-operations-and-troubleshooting.md) | Known behaviours, common pitfalls, secrets/rotation, re-deploy/reset, blueprint re-pinning |

## Authority order

When sources disagree, trust them in this order:

1. **The platform blueprint** (`blueprints/ARBITR_PP_SecOnion-*.yml`) — the
   definitive list of what VMs exist, their images, IPs and sizing.
2. **The inventory** (`hosts`) — group membership, which drives what Ansible
   does to each host.
3. **`group_vars` / `host_vars`** — the single source of truth for addressing,
   credentials, pins and tunables.
4. **`../../CLAUDE.md`** — the build contract and the hard-won pitfalls.
5. **This pack** — narrative and rationale over the above.

A number in prose is only as good as the blueprint/inventory it was derived
from. Where this pack states a count (72 hosts, 43 Windows, 41 Sysmon, …) it was
computed from the current blueprint and inventory; re-derive after any change
rather than trusting the text.

## Keeping this pack honest

- Describe **what is**, not what changed — build history lives in
  `../../UPSTREAM_FIXES.md` and the git log, not here.
- Every overlay workaround gets an `UPSTREAM_FIXES.md` entry the same change it
  lands in.
- `docs/` is **not** bundled into `ab_pp.tgz` and is not deployed, so editing
  this pack never requires a tarball rebuild.
