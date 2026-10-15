# PPSM Builder — Architecture

*For anyone changing the tool. Version 3.8.*

## Design constraints

Every design decision follows from the environment the tool runs in:

| Constraint | Consequence |
|---|---|
| Air-gapped, standalone workstations | One self-contained HTML file. No build step, no package manager, no external script, font or stylesheet. The file opens from `file://` and is verified by opening it with the network disconnected. |
| No software installation permitted | Vanilla JavaScript only. The `.xlsx` writer, the ZIP/DEFLATE codec, the PDF text extractor and the pcap parser are all implemented in the file rather than imported. |
| Mixed, undocumented inputs | Every file is identified from its **contents**, never its name or extension. A file that cannot be identified is reported, not guessed. |
| Reviewers return registers for specific defects | The review checks encode those defects; the eMASS export reproduces the official template byte-for-byte in structure so nothing is renamed, reordered or added. |
| Data must be traceable | Every row records its origin tools and source host. Nothing is written to the working register by an export. |

The browser keeps nothing between sessions except the Simple-mode toggle (`localStorage`). All working data lives in memory and in the session file the user saves.

## Data flow

```text
                     ┌──────────────────────────────────────────────────────────┐
  collectors         │                     ppsm-builder.html                    │
  ──────────         │                                                          │
  ppsm-collect.ps1 ─┐│  handleFiles()                                            │
  ppsm-collect.sh  ─┼┼─► read as text / bytes ─► parseFile(name, text)          │
  ppsm.yml         ─┘│        │                      │ content sniffing           │
  firewall / switch ─┤        │                      ▼                            │
  FMC / ACAS exports─┤        │   ┌──── traffic parsers ────┐  ┌── name sources ──┐│
  DNS / AD exports  ─┘        │   │ netstat, ss, collector  │  │ DNS zones, AD,   ││
                              │   │ CSV, show conn, FMC,    │  │ hosts, DHCP, ARP,││
                              │   │ NetFlow, Zeek, pcap,    │  │ CMDB, Nessus,    ││
                              │   │ nmap, Nessus, syslog    │  │ FMC hosts        ││
                              │   └───────────┬────────────┘  └────────┬─────────┘│
                              │               ▼                        ▼          │
                              │   ┌── rule parsers ──┐        name inventory      │
                              │   │ ASA ACL, FMC ACP │        (rebuildNameIndex)  │
                              │   │ Palo Alto,       │                 │          │
                              │   │ iptables, WFP    │                 │          │
                              │   └────────┬─────────┘                 │          │
                              │            ▼                           ▼          │
                              │   ROWS[] — one record per observed flow / rule    │
                              │                                                   │
                              │   finishRegister()                                │
                              │     1. applyHostmap      names ⇄ addresses        │
                              │     2. crossReference    rules ⇄ flows,           │
                              │                          scan services ⇄ traffic  │
                              │     3. enrichRows        derivation engine:       │
                              │                          direction, boundary,     │
                              │                          service, application,    │
                              │                          eMASS type, points       │
                              │     4. excludeEphemeral  ephemeral-port hygiene   │
                              │     5. dedupeRows        same flow → one row      │
                              │     6. consolidateRows   same src→dst, many ports │
                              │                                                   │
                              │   review: PPSM policy checks · rule coverage ·    │
                              │           diffRegisters (previous register)       │
                              │                                                   │
                              │   export: emassShape() ─► buildEmassXlsx()        │
                              │           register xlsx/csv · findings · coverage │
                              │           diff · endpoints · diagram · session    │
                              └──────────────────────────────────────────────────┘
```

Repeat sightings merge as they import: a flow that netstat, `show conn` and NetFlow all saw becomes one row whose Origin column lists all three.

## Module map

The file is organised into banner-commented sections. Line numbers drift; search for the banner text.

| Section | Responsibility | Key entry points |
|---|---|---|
| Styles and markup | Layout, cards, table, settings dialog, help panel | — |
| Service and application directories | Port → full service name (never a code); product name → common service and usual port | `guessService`, `guessServiceCode`, `serviceDisplayName` |
| Nessus / Tenable input | `.nessus` XML and Tenable CSV: listeners, netstat plugin connections, host properties, credentialed flag, **installed-software inventory** (plugins 20811, 22869, 45590, 44401) | `parseFile` → Nessus branch |
| Register table | Row model, rendering, inline editing, bulk-apply, dedupe | `blank`, `render`, `bulkControl`, `dedupeRows` |
| Ingest | Drag-and-drop, folder walking, encoding detection (UTF-8 ± BOM, UTF-16), content sniffing | `handleFiles`, `parseFile` |
| Configuration engine | Defaults, settings dialog, profile import/export | `CFG_DEFAULTS`, `CFG`, `openSettings`, `exportProfile` |
| Binary parsers | pcap / pcapng; DEFLATE inflate; ASCII85 | `inflateRaw`, `ascii85Decode` |
| Text parsers | `show conn`, NetFlow cache, Zeek, Windows Firewall log, tcpdump, ASA syslog, pktmon, iptables | `parseShowConn`, `parseNetflowCache`, `parseZeek`, … |
| Diagrams | Boundary flow map and environment diagram as SVG; PNG and print | `svgFlowMap`, `currentDiagramSvg` |
| Hostname resolution and endpoints | Name inventory with authority ordering; endpoint list; Unidentified Work List | `rebuildNameIndex`, `applyHostmap` |
| eMASS template | Shaping (one row per port, addresses only, port order, purpose on every row), workbook writer with Glossary/Instructions sheets, data validation, character limits, output file naming | `emassShape`, `buildEmassXlsx`, `emassMissing`, `emassFilename` |
| PDF text extraction | Stream inflation, font subset decoding, positioned text for tabular reports | `extractPdfText`, `pdfTextFromContent` |
| Derivation engine | Direction, boundary (Internal / B13 / B14 / interface-derived), enclave, service, application (including inventory match), eMASS Type and logical points, purpose drafts | `enrichRows`, `ENRICH_FIELDS` |
| PPSM policy layer | Must fix / Should fix / Worth a look checks; findings export | `renderPpsmChecks` |
| Firewall rules | Cisco ACL + object-groups, FMC ACP export, Palo Alto, `iptables-save`, `Get-NetFirewallRule`; rule ⇄ traffic cross-reference; coverage report | `parseCiscoAcl`, `parseAcpExport`, `crossReference` |
| Previous register diff | Explodes both sides to flows and compares | `diffRegisters` |
| FMC reports | Connection events, Servers, Hosts, Applications, VPN; ACP as PDF | `parseFmcReport`, `fmcEventFlows` |
| Excel input | Minimal `.xlsx` reader (shared strings, inline strings) | — |
| NetFlow analyser reports | Top Conversations workbooks | — |
| DNS and AD sources | BIND zones, `dnscmd`, DNS Manager exports, `Get-DnsServerResourceRecord` CSV, AD computer exports | — |
| Collector CSV | `<host>-connections.csv` written by the collectors; raw `Get-NetTCPConnection` exports | — |
| Consolidation | Ports per source→destination; sources per service | `consolidateRows` |
| Register exports | CSV / `.xlsx` writer (own ZIP + CRC32), printable report | `buildXlsx` |
| Classification banner | Cycles through markings; colours the bar | `bannerClassFor` |
| Self-test | 331 checks over parsers, derivation, dedupe, export | `runSelfTest` |
| Wiring | Button handlers, Simple mode, `finishRegister` | `finishRegister` |

## Row model

Each entry in `ROWS` is a flat object created by `blank()`. The columns the table shows are listed in `COLS`; the eMASS export maps them onto the template's 21 columns in `EMASS_HEADERS` order. Fields worth knowing:

| Field | Meaning |
|---|---|
| `src`, `srcHost`, `dest`, `dstHost` | Addresses and the names kept beside them. A hostname found in an address cell is moved beside it on resolution. |
| `proto`, `srcPort`, `dstPort` | Protocol and ports; `dstPort` may be a semicolon list after consolidation. |
| `direction`, `boundary`, `enclave`, `srcZone`, `dstZone` | Derived; `boundary` is `Internal only` or a B-number tag; zones are the firewall interface names where the data carried them. |
| `dataService`, `appName`, `banner` | Service (full name), product (Application/Software Record Name), version banner. |
| `origin`, `collector`, `pkts`, `bytes` | Which tool(s) saw it, which collected host the file came from, counters. |
| `status`, `isaRef` | New / Registered / permitted or denied by rule; the rule or agreement reference. |
| `justification`, `calRef`, `assurance`, `cal`, `registryId`, `subPath` | The judgement fields and PPSM registry bookkeeping. |
| `emassType`, `srcLoc`, `dstLoc`, `srcPoint`, `dstPoint`, `vpn`, `vpnType` | The eMASS-only columns: Type, physical locations, connection logical points, VPN. |

Rules and observations share the model but are never merged with each other.

## Extending the tool

**Adding an input format.** Add a sniffing test and parser branch in `parseFile`; return `{kind, rows, note}` using `blank()` for rows and set `origin`. Decide which end is the service (see *Which end is which* in the user guide). Add at least two self-tests: one that the format is recognised, one that direction and ports come out right.

**Adding a review check.** Add a function in the policy layer that returns the offending rows with a short reason, register it with its severity, and add a self-test with one passing and one failing row.

**Changing the eMASS export.** The template is reproduced exactly; do not add, rename or reorder columns. Shaping belongs in `emassShape`; workbook structure in `buildEmassXlsx`. Confirm the result opens in Excel with the dropdowns intact and that the self-tests covering the export still pass.

**Versioning.** Bump the `<title>`, the `.vtag` span, `VERSION`, and add a `CHANGELOG.md` entry. The self-test count in the documentation must match what the button reports.

## Testing

The **Self-Test** button runs every check in the browser; there is no external test tooling, so the suite runs the same on an air-gapped workstation as anywhere else. Add a test for every parser branch and derivation rule you touch — the checks are what make a 650 KB single file safe to change.

## Collectors

The collectors are independent of the builder and of each other. All three write the same results-folder layout (`hosts/`, `inventory/`, `network/`, `READ-ME.txt`), and the Windows and Ansible collectors write a `Collector` column recording the shell version and method used per host so the builder can show it. See [`../collectors/README.md`](../collectors/README.md).
