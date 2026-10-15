# Collectors

Three independent collectors gather the host, name and network evidence the builder reads. All of them write the same results folder, which is dropped onto the builder as a whole. None of them installs anything, changes anything on the hosts they read, or sends data anywhere but the results folder.

| Collector | Platform | Reach |
|---|---|---|
| [`windows/ppsm-collect.ps1`](windows/ppsm-collect.ps1) | Windows PowerShell 3.0–5.1, PowerShell 7 | Local host; other Windows hosts over WinRM in parallel; AD and DNS exports when the RSAT modules exist; Cisco ASA / IOS devices over SSH |
| [`linux/ppsm-collect.sh`](linux/ppsm-collect.sh) | bash 4.4+, iproute2 (RHEL 8/9 as shipped) | Local host; other Linux hosts over SSH |
| [`ansible/ppsm.yml`](ansible/ppsm.yml) | ansible-core 2.12+ | Whole fleet: Linux, Windows, Cisco firewalls and switches in one run |

**Run elevated.** Without Administrator or sudo the process behind each port is missing, and that is the field that proves a port is a real listening service rather than a passing connection.

The full procedure, including the firewall, switch, FMC and ACAS steps no script can take for you, is in [`../docs/user-guide.md`](../docs/user-guide.md) §4 (scripted) and §5 (manual).

---

## Output layout

```text
ppsm-results\
  READ-ME.txt                    when, by whom, which hosts were reached and which
                                 were not, which shell and method each host used
  hosts\<host>-connections.csv   one row per socket: source and destination
                                 address/port, protocol, state, process, PID, Collector
  hosts\<host>-netstat.txt       the raw netstat -ano, for reference
  hosts\<host>-sockets.txt       Linux: ss output with a "# ppsm-host:" header line
  hosts\pfirewall.log            the Windows Firewall log, if enabled
  inventory\ad-computers.csv     every computer account with an address
  inventory\dns-records.csv      A, PTR and CNAME records from every zone
  inventory\hosts-file.txt       the local hosts file
  inventory\arp-table.txt        Linux: ip neigh
  network\<device>-showconn.txt  the firewall connection table
  network\<device>-acl.txt       access-lists and object-groups
  network\<switch>-netflow.csv   the NetFlow cache
```

Files are named after the host they came from. That name is how a service listening on `0.0.0.0` is attributed to the right machine, so keep it when moving files by hand.

---

## Windows — `ppsm-collect.ps1`

```powershell
.\ppsm-collect.ps1                                    # this machine (plus AD and DNS on a DC)
.\ppsm-collect.ps1 -Computers dc01,file01,sql01
.\ppsm-collect.ps1 -ComputerListFile .\hosts.txt
.\ppsm-collect.ps1 -ComputerListFile .\hosts.txt -ThrottleLimit 64 -ProbeTimeout 3
.\ppsm-collect.ps1 -NetworkDevices fw01:asa,core01:ios -NetworkUser admin
.\ppsm-collect.ps1 -SelfTest
```

| Parameter | Default | Purpose |
|---|---|---|
| `-Computers` | local machine | Hosts to collect over WinRM |
| `-ComputerListFile` | — | One host per line; `#` lines ignored (see [`hosts.example.txt`](hosts.example.txt)) |
| `-OutputPath` | `.\ppsm-results` | Results folder |
| `-Credential` | current user | Run as another account |
| `-IncludeEstablished` | `$true` | `-IncludeEstablished:$false` for listeners only |
| `-SkipAD`, `-SkipDNS` | off | Skip the AD / DNS exports even where the modules exist |
| `-NetworkDevices` | — | `name[:type]` entries; `asa` (default) pulls `show conn all`, access-lists and object-groups; `ios` pulls the NetFlow cache |
| `-NetworkUser`, `-FlowMonitor` | —, `FLOW-IN-IQCORE` | SSH user for the devices; flow monitor name for `ios` |
| `-ThrottleLimit` | 32 | Hosts collected at the same time |
| `-ProbeTimeout` | 5 s | Wait for the WinRM port before a host is skipped |
| `-ConnectTimeout`, `-HostTimeout` | 20 s, 180 s | Session open / per-host completion limits |
| `-WinRmPort`, `-UseSSL` | 5985, off | 5986 with `-UseSSL` |
| `-SkipProbe` | off | Go straight to WinRM without the port check |
| `-SelfTest` | off | Collect the local machine normally and via the `netstat` fallback, compare, write `ppsm-selftest\SELFTEST.txt` |

**Reach and speed.** Every host's WinRM port is probed at once; anything that does not answer within `-ProbeTimeout` is listed as unreachable and skipped. Reachable hosts are collected `-ThrottleLimit` at a time. A host that answers but stalls is dropped after `-ConnectTimeout` / `-HostTimeout`. The run never waits on one host.

**PowerShell versions.** The control machine and the targets may differ. Each target checks its own `$PSVersionTable`: ports come from `Get-NetTCPConnection` / `Get-NetUDPEndpoint` where the NetTCPIP module exists, otherwise `netstat -ano` is parsed; OS name from `Get-CimInstance` on 3.0+, `Get-WmiObject` below; 5.1 writes UTF-8 with a BOM and 7 is told to (`utf8BOM`) so Excel opens both the same way. The method used is recorded per host on the console, in `READ-ME.txt`, and in the `Collector` column of each connections CSV. Run `-SelfTest` once on a 5.1 host before trusting a fleet run.

**Modules.** None required. RSAT ActiveDirectory adds the AD export and DnsServer adds the DNS zones; both are skipped cleanly when absent. The Windows OpenSSH client (`ssh.exe`, built in since Windows 10 1809 / Server 2019) is used for `-NetworkDevices`; you are prompted for the password unless a key is set up.

---

## Linux — `ppsm-collect.sh`

```bash
sudo ./ppsm-collect.sh                      # this host
sudo ./ppsm-collect.sh -o /tmp/ppsm         # choose the output folder
sudo ./ppsm-collect.sh -H hosts.txt         # also collect over SSH
sudo ./ppsm-collect.sh -H hosts.txt -u svc-ppsm
sudo ./ppsm-collect.sh -L                   # listeners only, no connections
```

| Option | Purpose |
|---|---|
| `-o <dir>` | Results folder (default `./ppsm-results`) |
| `-H <file>` | Host list, one per line, collected over SSH in the same run |
| `-u <user>` | SSH user for `-H` |
| `-L` | Listeners only |
| `-h` | Usage |

Uses `ss` and `ip` from iproute2; nothing depends on net-tools. Also copies `/etc/hosts` and the neighbour table into `inventory/` as extra name sources.

---

## Ansible — `ppsm.yml`

```bash
ansible-playbook -i inventory.ini ppsm.yml
ansible-playbook -i inventory.ini ppsm.yml -e results_dir=/tmp/ppsm
ansible-playbook -i inventory.ini ppsm.yml --limit dmz
```

Plays run against the inventory groups `linux`, `windows`, `cisco` (firewalls) and `switches`; groups absent from the inventory are skipped with a warning. Results are fetched to `results_dir` (default `./ppsm-results`) on the control node in the same layout as the other collectors. Start from [`ansible/inventory.example.ini`](ansible/inventory.example.ini).

Requires ansible-core 2.12+ plus the `ansible.windows` and `ansible.netcommon` collections and the `cisco.asa` / `cisco.ios` platforms you use. All are bundled in the full `ansible` package; on an air-gapped control node install them from downloaded tarballs with `ansible-galaxy collection install <file>.tar.gz`.

Windows hosts are read with `Get-NetTCPConnection` where available and `netstat -ano` otherwise; the `Collector` column in each connections CSV says which.

---

## Not in this repository

`ppsm-discover.ps1`, which sweeps address ranges and writes the `hosts.txt` the collectors read, is maintained separately.
