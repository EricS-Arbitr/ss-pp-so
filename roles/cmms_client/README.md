# cmms_client

Plants the **discoverable connection string** for the plant CMMS on the
engineering workstations. This is the other half of `roles/cmms`: that role
builds the maintenance database on `pp-sql` and creates the least-privilege
`svc_cmms` login; this role leaves that login's credential where an attacker
finds it.

## What it does

Drops a .NET application config —
`C:\Program Files\Voltgrid\CMMS Client\Voltgrid.CMMS.Client.exe.config` — on
every host in `[engineering]` (`pp-eng-wkstn-1..8`). Its `<connectionStrings>`
section carries a **plaintext SQL-auth** string:

```
Data Source=pp-sql.voltgrid.com;Initial Catalog=VoltgridCMMS;User ID=svc_cmms;Password=<vaulted>;...
```

A `Release Notes.txt` sits beside it so the folder reads like a real install
rather than a lone config file.

## The lesson

Endpoint → database. An attacker who lands on an engineer's workstation greps
the filesystem, finds a connection string in a config, and pivots to the CMMS
database on `pp-sql:1433` — no exploit, just a credential left in a file. The
string authenticates because `svc_cmms` is a real SQL login (mixed mode is on,
and `roles/cmms` opens 1433). What it yields is bounded on purpose: `svc_cmms`
holds `db_datareader`/`db_datawriter` on `VoltgridCMMS` and nothing else, so
the reward is the maintenance data — the asset register with its OT addresses
and the outage windows — not the instance. That asymmetry is the point, and it
is why `roles/cmms` created the login least-privilege.

## Credential handling

The password is **plaintext on the host by design** — that is the artifact. In
this repo it is only ever the vault reference `vault_cmms_app_password`, the
same value `roles/cmms` creates the login with, so the two stay in lock-step
and the planted string actually works. The template task carries `no_log`, so
the secret never reaches the deploy log.

## Keep in sync with roles/cmms

`cmms_client_db_name`, `cmms_client_app_login` and `cmms_client_app_password`
must match `roles/cmms`. If the database name or login changes there, change
them here too, or the planted string goes dead and the scenario path breaks.

## Run

```
ansible-playbook playbooks/00-baseline.yml --tags cmms_client --vault-password-file <file>
```

Idempotent: `win_template` rewrites the config only when its rendered content
changes.
