# PPSM Builder — Overview

*What it is and what it produces. Version 3.8.*

## The problem

A PPSM register has to say, for every service an enclave uses: what protocol and port, on which system, crossing which boundary, permitted by what, and why it needs to exist. The data for that is scattered across hosts, firewalls, switches, scanners and DNS, in a dozen formats that do not agree with one another. Assembling it by hand is slow and the result is hard to defend.

## What the builder does

It is a single HTML file, opened in a browser. Drop in whatever you have collected and it reads each file, works out what kind of file it is from the contents rather than the name, and cross-references every source against every other.

- **Names.** DNS zones (forward and reverse), AD, CMDB, hosts files, scans and discovery all feed one inventory, applied to every row: an address gains its hostname, a hostname gains its address, a short name is completed to the full one. Where sources disagree the most authoritative wins.
- **Rules against traffic.** A firewall ruleset says what is permitted; the flows say what happened. Matching them shows rules nobody uses and traffic no rule authorises — both findings in their own right.
- **One row per service.** The same flow seen by four tools becomes one row that lists all four, rather than four near-duplicates.
- **Derivation.** Boundary, enclave, data service, description, eMASS type and connection points are worked out from the data. Internal traffic is marked Internal and carries no B number.
- **Software inventory.** A credentialed ACAS scan lists what is installed on each host; the product that serves a port becomes its Application/Software Record Name.
- **Review checks.** The register is checked against what PPSM submissions actually get sent back for, before it goes anywhere.

## What you get out

| Output | What it is |
|---|---|
| `<System> - eMASS PPSM Template.xlsx` | The submission workbook, matching the official eMASS PPSM Boundary Information Import Template column for column, with its Glossary and Instructions sheets. Also available as `.csv`. |
| Register (`.xlsx` / `.csv`) | The full working register, including the columns eMASS does not take. |
| Boundary diagram (SVG / PNG) | Drawn from the register itself, not from a template. |
| Review findings (`.csv`) | Every outstanding item with the rows attached, to hand to system owners. |
| Coverage and diff reports (`.csv`) | Which rules the traffic used and which flows no rule permits; what changed since the last submission. |
| Session (`.json`) | Everything, including the host and software inventories, so work can be reopened and continued. |

## What it does not do

It does not certify compliance. The accuracy of the inputs, and the boundary and assurance analysis, remain the assessor's responsibility. Three fields are judgements rather than observations and no tool can fill them: the mission justification, the CAL or VA report reference, and the assurance category. The builder lists exactly which rows are missing each one.

It also never assigns a DISN boundary (B1–B8, B15) by itself. Nothing in host output, a firewall connection table or a flow cache can prove a DISN crossing.

## How it runs

Entirely in the browser, on the machine in front of you. Nothing is uploaded, nothing is installed, and there is no network call of any kind — so it is usable on a standalone or air-gapped workstation. The collector scripts (PowerShell, bash and Ansible) are equally self-contained and add nothing to the hosts they run on.

## The documents in this repository

| File | What it is for |
|---|---|
| `app/ppsm-builder.html` | The tool. |
| `docs/user-guide.md` | The complete guide: every collection step, every input, every setting and export. |
| `docs/sop.html` | The procedure as a browser page with tick-off cards for each collection step; printable. |
| `docs/architecture.md` | How the tool is put together, for anyone changing it. |
| `collectors/` | The collector scripts and their read-me. |
| `samples/` | A worked example covering every supported format. |
