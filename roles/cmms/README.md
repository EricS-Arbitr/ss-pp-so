# cmms — Voltgrid Power maintenance system of record

Installs SQL Server 2022 on `pp-sql` and builds `VoltgridCMMS`: the plant's
asset register, work-order history, PM schedules and **scheduled outage
windows**.

## Why it exists

`pp-sql` carried the name since the range was built and held no database.
`[sql2022]` sat unimplemented in `ACTION_PLAN.md` as item B5, and
`billing.voltgrid.com`'s data has always been SQLite on `pp-www`. This role
makes the name honest.

## Why a CMMS rather than moving billing here

A maintenance system is a better adversary target than a billing backend, for
reasons specific to a plant:

- **`outage_windows` names when a unit is off line.** A planned outage is the
  plant stating, in writing, when equipment is down and the crew is committed.
- **The asset register carries the OT addresses.** `DCS-01`, `PLC-GAS-01`,
  `VIB-MON-01` and `HIST-01` appear with their real IPs on this range, next to
  criticality ratings and notes like "keyswitch normally in RUN". An attacker
  reading an IT-segment database learns which OT host is worth reaching and
  why — that asymmetry is the lesson.
- **`svc_cmms` is a credential worth finding.** It owns this database and holds
  no server-level rights, so the connection string yields maintenance data, not
  the instance.

## Shape

| | |
|---|---|
| Engine | SQL Server 2022, default instance, TCP 1433 |
| Media | in-platform Nexus (`sql_server_installer`) — zero egress |
| Database | `VoltgridCMMS`, 7 tables |
| Reference data | curated: 25 assets, 12 technicians, 6 vendors, 20 parts, 24 PMs, 10 outages |
| Volume data | 420 work orders generated with `RAND(cmms_seed)` — reproducible across rebuilds |
| Credentials | vaulted (`vault_cmms_sa_password`, `vault_cmms_app_password`) |

## Idempotency

Every gate tests an outcome, never a previous task's `changed`:

- the engine step is skipped when the `MSSQLSERVER` service exists
- the database step is skipped when `dbo.work_orders` already has rows
- the row-count proof runs on **every** pass, seeded or skipped, so a database
  emptied between deploys is reported rather than assumed intact

Gating installation on a notify is what left Splunk installed-but-never-
initialised on airfield-stacked and hung a deploy for twenty hours. See
`UPSTREAM_FIXES.md` 2026-09-27.

## Verification by hand

```powershell
sqlcmd -S pp-sql -d VoltgridCMMS -Q "SELECT unit, starts_on, reason FROM outage_windows WHERE starts_on > GETDATE() ORDER BY starts_on"
```
