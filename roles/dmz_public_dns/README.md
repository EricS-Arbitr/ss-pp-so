# dmz_public_dns

Makes **pp-dmz-dns** the public authoritative nameserver for **voltgrid.com** —
the internet-facing view of the zone.

## Why
Previously the simulated internet (`is-inet`, unbound) *faked* authority for
voltgrid.com with `local-data`. This role gives the DMZ box a real
authoritative zone so that `is-inet` can **delegate** to it (unbound stub-zone),
and external DNS queries actually traverse the WAN NAT to pp-dmz-dns.

## What it does
- Installs the Windows **DNS Server** role on pp-dmz-dns (standalone, not a DC).
- Creates a **file-backed** primary `voltgrid.com` zone (`Add-DnsServerPrimaryZone
  -ZoneFile`) — not AD-integrated, because this host is not a domain member.
- Reconciles the **external-IP** records: `www`/`billing` → `75.21.1.1`
  (external-fw WAN → pp-www), apex `@` + `mail` → `52.96.223.2` (webmail),
  `MX 10 mail.voltgrid.com`, `ns1 → 75.21.1.1`, and a zone `NS → ns1.voltgrid.com`.
  Idempotent: a record already holding the right data is left alone.

## The other two moving parts (not in this role)
1. **pp-external-firewall** DNATs WAN `75.21.1.1:53` (udp+tcp) → `172.16.8.4`
   (in `host_vars/pp-external-firewall.yml` `pfsense_nat_rules`).
2. **is-inet** stub-delegates voltgrid.com to the WAN IP and stops answering it
   from `local-data` (the global_dns change).

## Split-horizon
The **internal** AD view of voltgrid.com stays on the domain controllers
(`roles/dns`, internal IPs). This role only owns the public view. Keep the
record values in sync with the voltgrid.com entries in
`group_vars/all/main.yml` `global_dns_records`.

## Deploy (existing range, no redeploy)
```
ansible-playbook site.yml --tags public_dns
```
Targets pp-dmz-dns, pp-external-firewall (NAT), and is-inet (delegation).
