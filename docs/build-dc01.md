# DC01: domain controller build

DC01 is the lab's only domain controller: Windows Server 2025 running Active Directory Domain Services and DNS for the domain `lab.local`. It is also the lab's time source. Every other VM depends on it for name resolution, authentication and time, so it is always started first.

Problems hit along the way are written up in [incident-reports.md](incident-reports.md); the numbers below (e.g. "incident 3") refer to that file.

## At a glance

| Area | Configuration |
|---|---|
| VM | Windows Server 2025 Standard (Desktop Experience), 2 vCPU, 4 GB RAM, 60 GB NVMe (thin-provisioned), UEFI |
| Network | VMware host-only, static 192.168.125.10/24, no gateway, DNS 127.0.0.1 |
| Domain | `lab.local` (NetBIOS `LAB`), new forest, functional level Windows Server 2025 |
| Roles | AD DS, DNS server, Global Catalog |
| Time | Domain time source, following the host PC's clock through VMware Tools |
| Licence | Evaluation, activated online; one rearm kept in reserve |

## 1. VM creation

- **Skip VMware's "Easy Install".** With the Windows Server evaluation ISO, Easy Install demanded a product key with no way past it. Choosing "I will install the operating system later" and attaching the ISO afterwards lets normal Windows Setup run, where "I don't have a product key" is available. The same workaround applied to the Windows 11 evaluation ISO for CLIENT01.
- **UEFI without Secure Boot or a TPM.** Windows Server doesn't require either; Windows 11 (CLIENT01) does.
- **Booting from the ISO:** the "press any key to boot from CD" prompt is easy to miss. VMware's **Power On to Firmware** (available when the VM is fully off) stops at the UEFI boot manager every time.

## 2. Network and name

- Static IP 192.168.125.10/24 with **no default gateway**: the host-only network has no route out, by design.
- **Preferred DNS 127.0.0.1**, i.e. the server itself. Active Directory depends on DNS (clients find domain controllers through DNS records), and DC01 hosts the `lab.local` zone, so it must ask itself.
- Renamed to **DC01** before promotion, so the domain controller is registered in AD and DNS under its final name.
- Checked with `ipconfig /all`: DHCP disabled, values as above.

## 3. AD DS role and promotion

- Server Manager > **Add Roles and Features** > Active Directory Domain Services (with its required management features).
- After the role installed: **Promote this server to a domain controller**:
  - New forest, root domain `lab.local`, NetBIOS name `LAB`
  - Forest and domain functional level Windows Server 2025
  - DNS server and Global Catalog on; not a read-only domain controller (RODC)
  - A separate **DSRM** (Directory Services Restore Mode) password. DSRM is the recovery mode used when AD itself isn't running, so it can't rely on a domain account and has its own local password.
- After promotion the domain administrator password was rejected, and access was recovered through the recovery environment (incident 3).

**On the name `lab.local`:** fine for an isolated lab, but not what I would choose for a real network. `.local` is reserved for multicast DNS (RFC 6762), and current guidance is to use a subdomain of a domain you own (for example `ad.example.com`).

## 4. Verification

- **`dcdiag`:** all core AD DS tests passed (Advertising, FrsEvent, SysVolCheck, KccEvent, KnowsOfRoleHolders, MachineAccount, NCSecDesc, NetLogons, Services, VerifyReferences, the partition tests, LocatorCheck, Intersite). Two warnings are expected in this lab and are not faults:
  - `DFSREvent`: SYSVOL replication warnings. With a single domain controller there is no replication partner.
  - `SystemLog`: DNS timeouts to Microsoft endpoints and "no external time source" warnings, because the network has no internet route.
- Server Manager: AD DS and DNS both healthy.
- PowerShell: `Get-ADDomain` and `Get-ADUser -Identity Administrator` both return the expected objects.

## 5. DNS

- `lab.local` A records checked: the zone root (`@`), `DomainDnsZones`, `ForestDnsZones` and `dc01` point to 192.168.125.10; `CLIENT01` points to 192.168.125.11.
- The default table view truncates the addresses. `-RRType A` plus a calculated column from `RecordData.IPv4Address` shows them in full, for example:
  ```powershell
  Get-DnsServerResourceRecord -ZoneName "lab.local" -RRType A |
    Select-Object HostName, @{Name="IPv4"; Expression={$_.RecordData.IPv4Address}}
  ```
  `@{Name=...; Expression=...}` is a calculated property: it adds a column built from a nested field.
- **No reverse lookup zone** for 192.168.125.0/24. Harmless, but it's why `nslookup` prints "DNS request timed out" and "Server: UnKnown" before every answer: it tries to look up the DNS server's own IP as a name first. Optional tidy-up: create the zone `125.168.192.in-addr.arpa`.

## 6. Licensing

- The Windows Server evaluation was found expired: a "Windows License is expired" watermark, and `slmgr /dlv` showed `Notification` with reason `0xC004F009` (grace time expired). An expired Server evaluation **shuts down every hour**, and on a domain controller that takes DNS, authentication and time away from every other VM.
- `slmgr /dlv` also showed **one rearm left**. Rather than spend it, I activated the evaluation online over a temporary NAT connection, with a snapshot first and the static settings recorded (incident 2). Result: Licensed for 180 days, rearm count still 1.
- Remaining plan: `slmgr /rearm` plus a reboot adds another 180 days when the current period ends.

## 7. Time

DC01 is the domain's time authority: CLIENT01 syncs from it through the Windows domain hierarchy, and ELASTIC01 through chrony.

- **Time zone:** GMT Standard Time. It had been left on Pacific time while the clock face showed UK time, which made the stored UTC eight hours wrong across the domain (incident 1). Fixed on DC01 first, then on CLIENT01:
  ```powershell
  Set-TimeZone -Id "GMT Standard Time"
  Set-Date -Adjust -8:00:00
  ```
  `Set-TimeZone` only changes how time is displayed; `Set-Date -Adjust` moves the actual clock, which is what corrects UTC.
- **Time source:** with no internet NTP on the host-only network, DC01 originally ran on its own clock with nothing correcting drift. I enabled VMware Tools' **"Synchronize guest time with host"**, so DC01 follows the host PC, which syncs from the internet. Measured from the host before and after with:
  ```powershell
  w32tm /stripchart /computer:192.168.125.10 /samples:3 /dataonly
  ```
  Offset went from 0.128 s to 0.0146 s, then 0.0068 s.
- **Deliberate trade-off:** production guidance is the opposite. The domain's top time source (the PDC emulator) should sync from external NTP, with hypervisor time sync disabled. In this isolated lab the host clock is the only accurate reference.
- VMware Tools updates are left on manual, so the domain controller never reinstalls or reboots unexpectedly.

## Known gaps

- Single domain controller: no redundancy, which is why the `DFSREvent` warnings exist.
- Time comes from the hypervisor, not from NTP (see above).
- No reverse lookup zone (optional).
- DC01 isn't yet sending its own logs to the SIEM. Onboarding it as a second endpoint is on the next-steps list in [build-elastic01.md](build-elastic01.md).
