# Sample inputs

Invented data, safe to drop on the builder to see it work. Nothing here describes a real system; every address, hostname and domain (`range.mil`, `range2.mil`) is fictitious.

Two sets:

- **`range2/`** — one coherent enclave seen by every tool at once. Drag the whole folder onto the Ingest box and the sources cross-reference each other: the firewall sets boundaries, DNS names the addresses, the ACAS scan names the products, the ruleset is matched to the traffic. This is the set to use for a demonstration or an end-to-end check.
- **`generic/`** — one file per format, including the awkward variants (ANSI colour codes in a captured session, a PDF with subset fonts, a space-separated Zeek log). Use these to check a single parser.

The builder identifies each file by **content**, never by name, so the names here are only descriptive. The one exception is host attribution: a per-host socket file is attributed to the host named in its file name or `# ppsm-host:` header.

## `range2/` — a complete enclave

| File | Format | What it gives the register |
|---|---|---|
| `range2-fw-showconn.txt` | ASA / FTD `show conn all` | Observed flows crossing the firewall, with the interface names that set Boundary |
| `range2-fw-showconn-subsetfont.pdf` | The same, printed to PDF with subset fonts | Exercises the PDF text extractor |
| `range2-asa-acl.txt` | `show running-config access-list` + object-groups | The ruleset; expanded through object-groups and matched to the flows |
| `range2-acp-export.csv` | FMC access control policy export | The same ruleset as FMC exports it |
| `range2-fmc-access-control-policy.pdf`, `range2-fmc-acp-rule-detail.pdf` | FMC policy report as PDF | Positioned-text PDF reading of the policy |
| `range2-fmc-connection-events.csv` | FMC Analysis → Connections → Events | Flows with initiator/responder, the rule behind each, applications |
| `range2-fmc-servers.csv` | FMC Analysis → Hosts → Servers | Discovered listening services with vendor and version |
| `range2-fmc-hosts.csv` | FMC Network Map | Hostnames and OS per address |
| `range2-fmc-intrusion-events.csv` | FMC intrusion events | Flows seen by the IPS |
| `range2-fmc-vpn.csv` | FMC Site-to-Site VPN report | VPN tunnel endpoints and types |
| `range2-netflow-cache.csv` | `show flow monitor <name> cache format csv` | East-west flows with input/output interfaces |
| `range2-netflow-top100-conversations.xlsx` | A NetFlow analyser's Top Conversations report | The same, as an analyser exports it (title rows above the header) |
| `range2-netflow-conversations-plixer.xlsx` | A second analyser layout, two sheets | Reader must find the data sheet |
| `range2-zeek-conn.log` | Zeek `conn.log` | Flows with `id.orig_h` / `id.resp_h` and the detected service |
| `range2-pfirewall.log` | Windows Firewall log | Host-level allow/drop events |
| `range2-acas-scan.nessus` | Nessus / ACAS export, 4 hosts | Listening services with product detection (22964), netstat connections, OS, FQDN/NetBIOS names, credentialed flag |
| `range2-ad-computers.csv` | `Get-ADComputer` export | Computer names, addresses, OS, owner |
| `range2-windows-dns-export.csv` | `Get-DnsServerResourceRecord` export (expanded record data) | A, PTR and CNAME records |
| `range2-dns-export.csv`, `range2-dns-export.txt` | Generic `name,type,data` DNS export, as CSV and as text | Forward records |
| `range2-dns-forward.zone`, `range2-dns-reverse-40.22.172.rev` | BIND forward and reverse zones | The authoritative name for every address |
| `range2-hosts.txt` | A hosts file | Local name overrides |

## `generic/` — one file per format

| File | Format | Note |
|---|---|---|
| `sample-show-conn.txt` | ASA `show conn all` | Baseline |
| `sample-show-conn-ansi.txt` | The same captured from a colour terminal | ANSI escape codes are stripped |
| `sample-show-conn-exported.pdf` | The same, printed to PDF | Plain PDF text |
| `sample-asa-syslog.log` | ASA syslog (302013 / 302015 "Built connection") | Flows with interface names |
| `sample-ufw.log` | Linux UFW kernel log | `[UFW ALLOW]` / `[UFW BLOCK]` events |
| `sample-pfirewall.log` | Windows Firewall log | — |
| `sample-tcpdump.txt` | `tcpdump` text output | Direction from SYN / SYN-ACK |
| `sample-capture.pcap`, `sample-capture.pcapng` | Packet captures | Binary parser, both container formats |
| `sample-zeek-conn.log` | Zeek `conn.log`, tab-separated with header block | Baseline |
| `sample-zeek-spaced.log` | Zeek fields, space-separated | Separator detection |
| `sample-zeek-exported.pdf`, `sample-zeek-subsetfont.pdf` | Zeek log printed to PDF | Plain and subset-font PDF text |
| `sample-netflow-cache.csv` | NetFlow cache as CSV | Without interface columns |
| `sample-netflow-cache.txt` | `show flow monitor … cache format table` | Fixed-width table form |
| `sample-nmap-scan.xml` | `nmap -sV -oX` | Open ports with product/version; closed ports excluded |
| `sample-ad-inventory.csv` | AD / CMDB export with an Enclave column | Names, addresses, enclave labels |
| `sample-dns-zone.txt` | BIND zone | Forward records |
| `sample-hosts.txt` | `/etc/hosts` | — |
| `sample-arp-table.txt` | `arp -a` / `ip neigh` | Lowest-authority name source |
| `sample-dhcpd.leases` | ISC `dhcpd.leases` | Address ⇄ hostname from leases |
| `sample-unidentified-worklist-FILLED.csv` | The builder's own Unidentified Work List, filled in by hand | Re-import of hostnames and owners typed by the system owner |

## Using the samples for a regression check

1. Open `app/ppsm-builder.html`, press **Self-Test** — all checks should pass.
2. Drag `samples/range2/` onto the Ingest box. The import log should identify every file (`show-conn`, `firewall ACL`, `FMC connection events`, `Nessus/ACAS`, `DNS zone`, `NetFlow top conversations (from Excel)`, and so on) with a non-zero entry or name count. Nothing should be reported as unreadable.
3. Press **Finish Register**. With Simple mode on this has already run on import; the register settles at roughly 28 rows after dedupe and consolidation. The review checks should list *Must fix* items for observed traffic with no rule, missing CAL/VA references and one scan-only service, and *Should fix* items for the judgement fields — those are the expected state of an unreviewed register, not parser faults.
4. **Export → eMASS PPSM Template (.xlsx)** and open the workbook in Excel: 21 columns on row 8, dropdowns intact on Boundary, Direction, Type, the logical point columns and the VPN columns.
