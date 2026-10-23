# 05 — Build and Deploy

## The deploy chain, end to end

```
build_tarball.sh  ──▶  ab_pp.tgz  ──▶  commit + push to GitHub (pinned commit)
                                               │
blueprint download-range-vars-script: wget ab_pp.tgz from the pinned raw URL
                                               │  (through mgmt proxy 10.255.240.1:3128)
                                   tar -xzf into /etc/ansible
                                               │
blueprint deploy-script:  ./deploy.sh | tee /var/log/playbook_run.log
                                               │
                                    site.yml, up to 3 attempts
```

The controller **deploys from the tarball, not a git pull**. The repo is just
where the tarball is built and versioned; the running range only ever sees
`ab_pp.tgz`.

## Building the tarball

```bash
./build_tarball.sh          # writes ab_pp.tgz; refuses if a gate fails
```

`build_tarball.sh` auto-discovers roles from **every** play under `playbooks/`
plus `site.yml` and `playbooks/00-baseline.yml`, understanding `roles:`,
`import_role` and `include_role` (so a role referenced only via
`import_role` + `tasks_from`, like `vyos_mirror`, is still bundled). Each role
is taken from `../range-development-ansible/roles/` first, then overridden by
the local `./roles/`. Bundled artifacts: roles, playbooks, `hosts`,
`group_vars`, `host_vars`, the verifier scripts, `deploy.sh`,
`requirements.yml`, the vendored `collections/`, and the ETOPEN `rules/`.
**Not** bundled: `docs/`, `blueprints/`.

It needs a `python3` with PyYAML. The system `/usr/bin/python3` has it; a
Homebrew `python3` may not — run `PATH="/usr/bin:$PATH" ./build_tarball.sh` if
the role-discovery step fails to import yaml.

### Build-time gates (the archive is refused if any fails)

| Gate | Catches |
|---|---|
| `verify_vars.py` | Jinja references with no definition, **and role scope** — a play referencing a role default without including that role (exits 3; build ABORTS, because it resolves in YAML and only fails on the range). Run it against the **stage**, not the repo root. |
| `verify_shell_args.py` | An apostrophe in a PowerShell/shell comment inside a free-form module arg — Ansible's `split_args()` counts quotes and the play fails to **load** (valid YAML, so a separate check). |
| `verify_so_inventory.py` | An SO host in `[so_all]` but not `[linux]` (roles to run, no way to log in). |
| `verify_dup_keys.py` | Duplicate YAML mapping keys (silently drop data). |
| `verify_handlers.py` | `notify:` targets that resolve to no handler. |
| `verify_task_keywords.py` | Task keywords misindented as module arguments. |
| `verify_defend_filters.py` | Malformed Elastic Defend filters. |

**Expected `verify_vars.py` warnings against the stage: 4** — intentional
`| default(...)` references and base-role vars present only in the stage
(e.g. `pfsense_stale_gateways`, `nat`, `splunk_admin_password`, and one of
`billing_secret_key` / `dns_zone_replication`). A warning count other than 4
means something changed — investigate before shipping.

## Publishing

Per the tarball-commit policy: rebuild `ab_pp.tgz` **and** commit it with the
code change, every time, because the controller deploys the tarball. After
committing, **verify the committed artifact, not the working copy**:

```bash
git show HEAD:ab_pp.tgz | tar tz | head        # it is really in the commit
git show HEAD:ab_pp.tgz | tar xzO ./hosts | grep -c <thing>   # spot-check content
```

Hand over a **commit-pinned** raw URL (not a branch URL — branch URLs are
CDN-cached):
`https://raw.githubusercontent.com/EricS-Arbitr/ss-pp-so/<full-sha>/ab_pp.tgz`

Then point the blueprint's `download-range-vars-script` at that SHA (see
[08-operations-and-troubleshooting.md](08-operations-and-troubleshooting.md)).

## `deploy.sh` — the 3-attempt retry model

`deploy.sh` runs `site.yml` up to three times, and **attempt 2 is not a
success condition**:

1. full sweep
2. retry-file scope — a **repair pass** over the hosts that failed in attempt 1
3. full sweep — **this is what actually confirms the range**

The retry file lists hosts that *failed*. Repairing them does not re-run the
plays whose targets were dropped when they failed, so a clean repair pass is not
a deployed range. Attempt 3 is the confirmation. A first-attempt failure is
often ordering (a host depends on state another host reaches asynchronously),
not a fault.

- `FORKS` is **derived**, not chosen: the largest single play target plus a
  small margin, so the widest play runs in one batch. Recount when hosts are
  added.
- `BOOT_DELAY` (default 180s; `BOOT_DELAY=0` to skip) lets a freshly
  provisioned range finish booting — it covers hosts that do not exist yet,
  which `wait_for_connection` cannot. Use `BOOT_DELAY=0 ./deploy.sh` on an
  already-up range.

```bash
sudo ./deploy.sh                 # full build, 3 attempts
BOOT_DELAY=0 ./deploy.sh         # already-up range
ansible-playbook playbooks/40-manager.yml            # one phase, during dev
ansible-playbook site.yml --limit pp-dc01,so-manager # targeted re-run
```

## Phase order (`site.yml`)

```
10-mirror           GRE/tc mirror build — runs FIRST (fails fast; kicks the
                    7 GB registry build off in the background and returns)
12-nexus-apt        apt-from-Nexus cleanup (base apt stays on the Ubuntu
                    archive; see "Artifact sourcing" below)
00-baseline         the range baseline: init + common across all hosts, AD
                    (forest + 3 DCs), CA, DNS, DHCP, firewalls, routers, corp
05-time             Windows clock, DC-first
20-vyos             GRE tunnels + tc mirror rules
30-prereqs          so_base on all SO nodes
40-manager          SO manager install — MUST complete before 50
50-nodes            search + sensor grid-join
60-verify           grid verification
70-analyst          analyst-workstation enablement
75-endpoint         Sysmon, then Elastic Agent enrolment
80-fleet-integrations  move log-source hosts onto dedicated Fleet policies
90-malcolm          Malcolm OT appliance
```

`10-mirror` is hoisted ahead of the baseline because the mirror/registry is a
hard prerequisite for 30/40/50 — discovering it broken after a multi-hour
baseline wastes the run — and because it overlaps the 7 GB image build with the
baseline. Safe to hoist because the controller is in
`[ansible_controller]`/`[infrastructure]` only, never `[linux]`.

`80-fleet-integrations` **must** follow `75-endpoint`: enrolment puts every
endpoint on `endpoints-initial`, then that phase reassigns the log-source hosts
to dedicated policies. Run it first and there is no agent to reassign.

## Secrets / vault

Every credential lives in `group_vars/all/vault.yml` (ansible-vault).
`ansible.cfg` points `vault_password_file` at `/home/simspace/.vault_pass` on
the controller, which **does not persist across range spin-ups** — `deploy.sh`
fails closed with the recreate command when it is missing. The repo is public:
never write the vault password into it; the password file and `*.pem/*.p12/
id_rsa*` are gitignored.

## Artifact sourcing (Nexus)

The range has no general egress. Everything is fetched from the in-platform
Nexus (`nexus.dev.ng.simspace.lan`) through the management proxy
(`10.255.240.1:3128`); Nexus presents an internal-CA cert, so each fetcher is
told to trust or skip-verify that host:

| Artifact | Nexus repo | Fetcher / note |
|---|---|---|
| SO source snapshot + airgap content | `ng_raw` (`installers/so/…`) | `get_url`, `validate_certs: false` |
| SO + Malcolm container images | `nexus-docker` | `skopeo --src-tls-verify=false`; dockerd `insecure-registries` |
| Malcolm 26.08.0 source | `ng_raw` (`installers/malcolm/…`) | `get_url` |
| Ansible Galaxy collections | `ng_raw/galaxy` | `ansible-galaxy … --ignore-certs` (set in `deploy.sh`) |
| Billing Python packages | `ng_pypi` | `pip --index-url … --trusted-host …` |
| **apt packages + `docker-ce`** | **Ubuntu archive / `download.docker.com`** | **NOT Nexus.** `ng-apt` serves jammy+noble+plucky from one index; no pin can safely prefer the right release, so apt stays on upstream through the proxy. Sourcing apt from Nexus needs per-release repos. |

Before cutting a blueprint over to a Nexus-sourced tarball, confirm Nexus is
populated and reachable through the proxy for that range — otherwise the deploy
fails early on the SO source or a Galaxy fetch.
