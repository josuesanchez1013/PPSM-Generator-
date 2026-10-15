# PPSM Builder

**Offline toolkit that turns raw host, firewall, switch, scanner and DNS output into a submission-ready eMASS PPSM register.**

Version 3.8 · single-file browser application + PowerShell, bash and Ansible collectors · no installation, no server, no network access · built for standalone and air-gapped systems.

---

## What it does

A Ports, Protocols and Services Management (PPSM) register has to state, for every service an enclave uses: the protocol and port, the system it runs on, the boundary it crosses, the rule that permits it, and why it needs to exist. That evidence lives in a dozen tools and formats that do not agree with each other.

PPSM Builder collects that evidence, cross-references every source against every other, and writes the **eMASS PPSM Boundary Information Import Template** exactly as eMASS expects it — 21 columns, header block, Glossary and Instructions sheets — as `<System> - eMASS PPSM Template.xlsx`.

| Capability | Summary |
|---|---|
| **Collect** | Parallel WinRM collection on Windows (PowerShell 3.0–7), SSH collection on Linux, or a single Ansible run across Windows, Linux and Cisco devices. Firewall and switch output can be fetched over SSH by the Windows collector. |
| **Ingest** | Dozens of input formats identified by *content*, never by file name: `netstat`/`ss`, `Get-NetTCPConnection`, ASA `show conn` and ACLs, FMC reports (CSV and PDF), Palo Alto policy, NetFlow caches and analyser reports, Zeek, pcap/pcapng, nmap XML, Nessus/ACAS (`.nessus` and Tenable CSV), AD and DNS exports, BIND zones, hosts files, DHCP leases, ARP tables, CMDB exports, previous registers. CSV, Excel, text and text-based PDF. |
| **Cross-reference** | One name inventory applied to every row in both directions; rules matched to the flows that used them; scan-discovered services folded into observed traffic; repeat sightings merged with their origin tools recorded. |
| **Derive** | Direction, boundary, enclave, Data Service (full names, never codes), Application/Software Record Name (from scan service detection and the installed-software inventory), eMASS Type and connection logical points. Internal traffic is marked Internal and never receives a DISN boundary. |
| **Review** | Checks graded *Must fix* / *Should fix* / *Worth a look* against what PPSM submissions are actually returned for; rule-coverage analysis; diff against the previously submitted register. |
| **Export** | eMASS PPSM Template (`.xlsx` / `.csv`), full register, findings, coverage and diff reports, endpoint list, boundary diagram (SVG/PNG), printable report, session and settings profiles. |
| **Verify** | 331 built-in self-tests run from the **Self-Test** button, in the browser, with no tooling. |

What it does **not** do: certify compliance, assign DISN boundaries (B1–B8, B15), or write the three judgement fields — Mission Justification, CAL/VA Reference and Assurance Category — for you. It tells you exactly which rows still need them.

---

## Quick start

**1. Collect** (run elevated — without Administrator/sudo the process behind each port is missing)

```powershell
# Windows — one domain-joined machine reaches the rest over WinRM
.\ppsm-collect.ps1 -ComputerListFile hosts.txt

# ...and once on a domain controller, to export every computer account and DNS zone
.\ppsm-collect.ps1
```

```bash
# Linux — this host, or the others over SSH in the same run
sudo ./ppsm-collect.sh -H hosts.txt
```

```bash
# Whole fleet
ansible-playbook -i inventory.ini ppsm.yml
```

Add the firewall connection table and ruleset (`show conn all`, `show running-config access-list`, `show running-config object-group`) to `ppsm-results\network\`, or let the Windows collector fetch them with `-NetworkDevices fw01:asa -NetworkUser <user>`. Export the ACAS scan as `.nessus` into `ppsm-results\`.

**2. Build** — open `app/ppsm-builder.html` in any current browser. Under Settings set the enclave location, DNS domain and Internal ranges (if not RFC1918). Drag the whole `ppsm-results` folder onto the Ingest box. Press **Finish Register**.

**3. Review** — resolve hostnames the Endpoints panel says have no address; run the review checks and clear every *Must fix*.

**4. Submit** — **Export → eMASS PPSM Template (.xlsx)**, then **Save Session** before closing the tab.

Try it first with the invented data in [`samples/`](samples/README.md).

---

## Repository layout

```text
ppsm-builder/
├── app/
│   └── ppsm-builder.html          The tool — one self-contained file, no dependencies
├── collectors/
│   ├── windows/ppsm-collect.ps1   Windows collector (WinRM, AD, DNS, network devices over SSH)
│   ├── linux/ppsm-collect.sh      Linux collector (ss/ip, SSH fan-out)
│   ├── ansible/ppsm.yml           Fleet playbook + inventory.example.ini
│   └── hosts.example.txt          Host-list format for the collectors
├── docs/
│   ├── overview.md                What it is and what it produces (two-minute read)
│   ├── user-guide.md              The complete guide: collection, ingest, review, export, settings
│   ├── architecture.md            How the builder is put together; data flow; module map
│   └── sop.html                   Interactive Standard Operating Procedure with tick-off cards
├── samples/                       Invented inputs for every supported format (see samples/README.md)
├── CHANGELOG.md                   Release history
└── VERSION                        Current version
```

---

## Requirements

| Component | Requirement |
|---|---|
| Builder | Any current browser opened from a local file (`file://`): Chrome, Edge, Firefox on Windows 10/11; Firefox or Chromium on RHEL 8/9. No plugins, no server, no internet. |
| Windows collector | Windows PowerShell 3.0–5.1 or PowerShell 7 on the collecting machine and the targets (they may differ). No modules required; RSAT ActiveDirectory and DnsServer are used when present. Windows OpenSSH client for `-NetworkDevices`. |
| Linux collector | bash 4.4+ and iproute2 (`ss`, `ip`). |
| Ansible playbook | ansible-core 2.12+, `ansible.windows`, `ansible.netcommon`, and the `cisco.asa` / `cisco.ios` platforms in use. |

---

## Security and data handling

- The builder makes **no network calls**. It has no external scripts, fonts or stylesheets, and can be verified by opening it with the network disconnected.
- The browser persists nothing about your data (only the Simple-mode toggle). Work is kept in the session file you save (`.json`), which contains every row, setting and inventory — treat it at the classification of the data it holds.
- The collectors write only to their results folder and change nothing on the hosts they read.
- The builder shows a classification banner (click it to cycle); the eMASS workbook keeps the template's own CUI banner block.
- Inputs are parsed as text, CSV, XML, ZIP/XML (`.xlsx`), PDF streams and pcap bytes; nothing in an input is evaluated as code, and every value rendered to the page or written to a workbook is escaped.
- The results folder, session files and exports describe your enclave's services, addresses, rules and installed software. Treat them at the classification of the system they describe; session files are plain-text JSON.

---

## Documentation

| Document | Read it when |
|---|---|
| [`docs/overview.md`](docs/overview.md) | You need to explain the tool to someone in two minutes. |
| [`docs/user-guide.md`](docs/user-guide.md) | You are running a collection, reviewing a register or shaping the export. |
| [`docs/sop.html`](docs/sop.html) | You want the procedure as tick-off cards, printable, for the person doing the work. |
| [`docs/architecture.md`](docs/architecture.md) | You are changing the tool. |
| [`collectors/README.md`](collectors/README.md) | You are running or modifying a collector. |
| [`samples/README.md`](samples/README.md) | You want to know what each sample file exercises. |
| [`CHANGELOG.md`](CHANGELOG.md) | You want to know what changed between versions. |

---

## Versioning and changes

Releases follow semantic versioning (major.minor.patch). The version is carried in `VERSION`, in the builder's title bar, and in `CHANGELOG.md`.

Rules for any change, so the tool stays usable where it is used:

- **No dependencies, no network calls, no build step.** The builder stays one HTML file that works from `file://`; the collectors use only what ships with the platform.
- **Inputs are identified by content**, never by file name or extension.
- **The eMASS template is reproduced exactly.** Columns and sheets are never added, renamed or reordered.
- **Nothing is guessed.** A value that cannot be defended to an assessor becomes a review check, not a value. DISN boundaries are never assigned automatically.
- **Export never changes the register.**
- Every parser branch or derivation rule touched gets a self-test in `runSelfTest()`; the count reported by the button must match the one in the documentation.
- Each change gets a line under **Unreleased** in `CHANGELOG.md`, written as what the user will see differently.

Before trusting a change: press **Self-Test** in Chrome/Edge and Firefox, drag `samples/range2/` onto the builder and confirm every file is identified, and open an exported eMASS workbook in Excel to check the 21 columns and dropdowns. If you touched a collector, run `ppsm-collect.ps1 -SelfTest` on a Windows PowerShell 5.1 host and `bash -n` / `shellcheck` on the Linux collector.

When reporting a problem, run **Self-Test** first: all green means the tool is fine and the input is the problem. Include the version, the browser, the failing check names, and an *invented* input that reproduces it — never real enclave data.

This is an internal tool. No licence is granted for use outside the owning organisation unless stated otherwise.
