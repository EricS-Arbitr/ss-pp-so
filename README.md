# ss-pp-so

Range-specific Ansible overlay for the **PowerPlant** cyber-range scenario
(`voltgrid.com`) on the SimSpace NG platform, with **Security Onion 2.4** as
the range's SIEM.

This is an overlay, not a standalone playbook. It layers on the customer's
shared platform repo at `../range-development-ansible/`: that repo ships base
roles, this one ships the range's inventory, variables, custom roles and
playbooks. `build_tarball.sh` combines the two into `ab_pp.tgz`, which is
extracted to `/etc/ansible` on the range's Ansible controller.

## Build and deploy

```bash
./build_tarball.sh          # -> ab_pp.tgz  (validates before it writes)
# copy ab_pp.tgz to the controller, extract to /etc/ansible, then:
./deploy.sh                 # site.yml, up to 3 attempts
```

`build_tarball.sh` refuses to write an archive that would not deploy. It
checks Jinja variable references (`verify_vars.py`), free-form shell arguments
that Ansible's `split_args()` would reject (`verify_shell_args.py`), and
Security Onion inventory membership (`verify_so_inventory.py`).

## What gets deployed

`site.yml` is the entry point. The SO mirror runs first so its container-image
build overlaps the range baseline, and because a broken mirror should fail in
minutes rather than after a multi-hour baseline.

| Phase | Does |
|---|---|
| `10-mirror` | nginx on the controller serving the SO source, detection content and container registry artifacts |
| `arbitr_pp_playbook.yaml` | the range baseline — network, AD, hosts, services |
| `05-time` | Windows clock correction, DC-first |
| `20-vyos` | GRE tunnels + `tc` mirror rules to the sensors |
| `30-prereqs` | `so_base` on every grid node |
| `40-manager` | waits for the registry artifacts, then installs the manager |
| `50-nodes` | search + sensors join the grid |
| `60-verify` | grid health |
| `70-analyst` | analyst workstation enablement |
| `75-endpoint` | Sysmon, then Elastic Agent enrolment into Fleet |
| `80-fleet-integrations` | host-scoped log sources into SO via dedicated Fleet policies |

## Security Onion

A distributed grid: manager, search node and three sensors. Endpoints report
through Elastic Agent; the sensors see network traffic through GRE mirrors
from the VyOS routers.

Container images are served from the controller's mirror rather than pulled
from ghcr.io, so no in-play system needs internet access. The controller is
the only host that reaches out, and only to build those artifacts.

## Documentation

| File | For |
|---|---|
| `CLAUDE.md` | working on this repo — layout, conventions, pitfalls, recipes |
| `docs/RANGE_GUIDE.md` | operators and instructors |
| `docs/RANGE_OVERVIEW.md` | the range at a glance |
| `UPSTREAM_FIXES.md` | customer-repo bugs and the overlay workarounds, with the reasoning |
| `PROJECT_LOG.md` | chronological build history |

Read `UPSTREAM_FIXES.md` before changing network, AD or Security Onion
plumbing. Most surprises are already diagnosed there.

## Secrets

Credentials live in `group_vars/all/vault.yml`, encrypted with ansible-vault.
`ansible.cfg` points `vault_password_file` at `/home/simspace/.vault_pass` on
the controller; that file does not persist across range spin-ups, and
`deploy.sh` fails closed with the recreate command when it is missing.
