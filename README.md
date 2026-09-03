# Enterprise Active Directory Lab (Project 1)

A self-built Active Directory environment simulating a small enterprise (`divine.lab`), designed and deployed entirely in VirtualBox. This project covers domain infrastructure, organizational unit design, tiered administrative security, Group Policy enforcement, shared resource permissioning, and fine-grained password policies.

## Environment

| Component | Details |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Domain Controller | Windows Server 2022 (`Berlin-DC-01`) |
| Domain | `divine.lab` |
| Client Machine | Windows Server 2022 (`ENPAL-SRV01`), domain-joined as a workstation |
| Network | Internal Network (isolated lab, static IPs) |

## Organizational Unit Structure

```
divine.lab
└── Berlin_Office
    ├── Divine Employees        (staging OU — new hires prior to department assignment)
    ├── Departments
    │   ├── IT                  → Lina Williams (Head of IT), Ella Ishimwe, Landry Kelly
    │   ├── HR                  → Danny Rita
    │   ├── Sales                (currently unstaffed — reserved for future growth)
    │   └── Finance              → Calvin Jose
    ├── Admins                  → lina-admin (tiered privileged account)
    ├── Computers
    │   ├── Workstations         → ENPAL-SRV01
    │   └── Servers               (reserved — DC itself sits outside this OU)
    ├── Groups                  → IT-Workstation-Admins, HR-Staff
    └── Service Accounts         (reserved for planned gMSA deployment)
```

**Design rationale:**
- **Site-based root (`Berlin_Office`)** mirrors how multi-branch organizations structure AD around physical/logical locations, not just function.
- **`Admins` is separate from `Departments`** so security policies targeting privileged accounts never accidentally apply to (or exempt) regular staff.
- **Computers are split from Users** since GPOs frequently need to target machines and people differently.
- **`Divine Employees` is retained as a staging OU** — new hires land here first, then get moved into their department OU as part of onboarding, mirroring a real HR/IT provisioning workflow.

## Users & Departments

| User | Department | Notes |
|---|---|---|
| Lina Williams | IT | Head of IT — see Tiered Administration below |
| Ella Ishimwe | IT | |
| Landry Kelly | IT | IT Support |
| Danny Rita | HR | Receptionist; logon-hours restricted; primary shared-folder test account |
| Calvin Jose | Finance | |

## Tiered Administration Model

Rather than granting Lina Williams elevated rights on her everyday account, this lab implements Microsoft's tiered administration pattern:

- **`lina.williams`** — standard daily-use account, no elevated privileges, used for email/normal work
- **`lina-admin`** — separate privileged account, placed in the `Admins` OU, added to the `IT-Workstation-Admins` security group

**Why this matters:** if the everyday account is ever phished or compromised, the attacker does not automatically gain administrative reach — that privilege lives entirely on a separate identity that isn't used for daily browsing/email.

`IT-Workstation-Admins` is granted local administrator rights on all computers in the `Workstations` OU via a linked GPO (Group Policy Preferences → Local Users and Groups), so any account added to this group automatically receives the correct access — no manual per-machine configuration needed.

## Group Policy Objects

| GPO | Scope | Function |
|---|---|---|
| Default Domain Policy | Domain-wide | Account lockout policy |
| GPO – Workstation Local Admin Rights | Workstations OU | Grants `IT-Workstation-Admins` local admin via GPP |
| GPO-Map-HR-Drive | HR OU | Auto-maps `H:` drive to `\\Berlin-DC-01\HR_Data` |
| Logon Hours Restriction | Applied per-account | Restricts standard staff logon windows (verified via Danny Rita's account) |
| Control Panel Restriction | (scope as configured) | Blocks endpoint access to Control Panel |

## Shared Folder: HR_Data

Implemented using the **AGDLP pattern** (Accounts → Global Group → Domain Local Permissions) rather than assigning permissions to individual users:

1. `HR-Staff` security group created; Danny Rita added as a member
2. `HR_Data` folder created on the Domain Controller (`C:\HR_Data`)
3. **Share permissions**: `HR-Staff` granted Change + Read (Everyone removed)
4. **NTFS permissions**: `HR-Staff` granted Modify + Read & Execute
5. Drive auto-mapped to `H:` via GPO on domain login
6. **Verified**: logged in as Danny Rita, confirmed `H:` drive auto-mounted, created `HR_Test_Rita.txt`, confirmed the file physically landed in `C:\HR_Data` on the server

This approach means future HR hires only need to be added to `HR-Staff` to receive correct access — no per-user permission configuration required.

## Password & Account Security Policies

- **Account lockout policy** — configured via Default Domain Policy
- **Fine-grained password policy (PSO)** — `PSO-Admin-Password-Policy` applied specifically to `IT-Workstation-Admins`, enforcing stricter requirements on privileged accounts than standard staff
- **Logon hours restriction** — verified functioning correctly by observing Danny Rita being blocked from authentication outside permitted hours

## Development Timeline

A running journal of how this project was actually built, session by session.

**Day 1 — Employee Onboarding Process**
- Created three departmental security groups: IT, Finance, Administration
- Created the `Divine Employees` OU within `Berlin_Office`
- Added five employee accounts and assigned each to their department
- Verified all accounts were created and correctly assigned

**Day 2 — Identity Management & Account Security**
- Assigned Lina Williams as IT Manager; linked IT employees under her to reflect the reporting structure
- Appointed Divioline Boss as CEO
- Configured logon hour restrictions: IT staff 06:00–18:00, other departments 08:00–18:00
- Configured account lockout policy: 5 failed attempts, 5-minute lockout duration, 15-minute counter reset

**Following weeks — Full build-out**
- Domain and client join, verified via `whoami`
- Full OU hierarchy design and user redistribution into department OUs
- Tiered administration model implemented (`lina-admin`, `IT-Workstation-Admins`)
- GPO deployment: workstation local admin rights, drive mapping, logon hours, lockout
- Fine-grained password policy (PSO) applied to privileged accounts
- Shared folder (`HR_Data`) rebuilt with proper AGDLP permissioning and end-to-end verified
- Extensive infrastructure troubleshooting (see Troubleshooting Log below) — clock synchronization, domain trust, boot order, and GPO application issues all diagnosed and resolved

## Troubleshooting Log

This section documents real issues encountered during the build — included deliberately, since diagnosing and resolving genuine infrastructure problems is a core sysadmin skill this project is meant to demonstrate.

| Issue | Root Cause | Resolution |
|---|---|---|
| Client stuck attempting PXE/network boot | VirtualBox boot order had Network above Hard Disk | Reordered boot priority in VM Settings → System → Motherboard |
| "We can't sign you in — your domain isn't available" | Multiple causes across sessions: clock drift, and later confirmed to be a network-reachability symptom, not credentials | Diagnosed via `ping`, `nslookup`, and `nltest /sc_query` to isolate DNS vs. secure channel vs. time issues |
| `gpupdate /force` failing with clock-sync error | Client and DC clocks differed by more than Kerberos' 5-minute tolerance, caused by VirtualBox syncing VM clocks to host time after Guest Additions installation | Manually corrected time via `Set-Date`, later diagnosed as a recurring VM quirk; adopted a habit of verifying `Get-Date` at the start of each session |
| VM freezing after clock changes / settings changes | Display/driver hang, likely tied to Kerberos ticket re-validation during abrupt clock jumps | Graceful shutdown signal via VirtualBox, or hard power-off as last resort, followed by clean restart |
| "Add Group..." greyed out in Restricted Groups editor | Unclear MMC snap-in state (possible stale session) | Switched to Group Policy Preferences (Local Users and Groups) instead — also the more modern, non-destructive method vs. Restricted Groups |
| GPO not appearing in `gpresult` output at all | Clock-sync error was blocking policy evaluation entirely before the GPO link/scope could even be checked | Resolved once clock sync was fixed; GPO applied correctly afterward |
| `net use` failing with "System error 67: network name cannot be found" | `HR_Data` folder had been deleted during earlier iteration of the shared-folder design | Recreated the folder with proper group-based AGDLP permissions |
| VM appeared to log in as local machine account instead of domain | `whoami` showed `machinename\administrator` instead of `domain\username` — misread of "which account is logged in" | Clarified distinction between local and domain identities; verified correctly afterward via `whoami` showing `divine\username` |

## Known Simplifications / Future Enhancements

- `Sales` OU is currently unstaffed — intentional, not an oversight, reflecting realistic small-org variation
- `Service Accounts` OU is currently empty — a gMSA (`LondonSQLFarm`) was researched and the KDS Root Key was initialized (`Add-KdsRootKey`) as groundwork, but the account itself has not yet been provisioned
- Windows LAPS (Local Administrator Password Solution) was researched as a natural extension of the tiered-admin work in this lab, but not yet implemented
- `HR_Data` is hosted directly on `C:\` for lab simplicity; a production environment would typically use a dedicated data volume

## Skills Demonstrated

- Active Directory Domain Services deployment and domain design
- Organizational Unit architecture for a multi-department organization
- Tiered administration / privileged access management
- Group Policy Object design (Restricted Groups vs. Group Policy Preferences)
- AGDLP permissioning model for shared resources
- Fine-grained password policies (PSOs)
- Kerberos authentication troubleshooting (clock synchronization)
- DNS and secure-channel diagnostics (`nslookup`, `nltest`, `ping`)
- Systematic, isolate-one-variable-at-a-time troubleshooting methodology


# Windows Infrastructure Lab (Project 2)

A second-server infrastructure build for the `divine.lab` domain, providing DHCP, DNS support, File Server, and Backup services deliberately separated from the Domain Controller to reflect real-world role separation rather than piling every service onto one box.

## Environment

| Component | Details |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Domain Controller (existing) | Windows Server 2022 (`Berlin-DC-01`) — AD DS + DNS |
| Infrastructure Server (new) | Windows Server 2022 (`BERLIN-FS-01`, VM name `FileServer03`) — DHCP, File Server, Backup |
| Domain | `divine.lab` |
| Network | Internal Network (isolated lab, static IPs) |
| BERLIN-FS-01 static IP | `192.168.1.20` |

**Design rationale:** DNS remains on the Domain Controller since it's tightly coupled with AD. DHCP, File Server, and Backup roles live on a separate member server reflecting the real-world practice of keeping Domain Controllers lean and dedicated to identity services, rather than overloading them with unrelated infrastructure roles.

## DHCP Server

**Role installation:** Installed via Server Manager → Add Roles and Features → DHCP Server, then authorized in Active Directory via PowerShell:
```powershell
Add-DhcpServerInDC -DnsName "BERLIN-FS-01.divine.lab" -IPAddress 192.168.1.20
```

**Scope configuration:**

| Setting | Value |
|---|---|
| Scope name | `Berlin_Clients_Scope` |
| Range | `192.168.1.50` – `192.168.1.150` |
| Subnet mask | `255.255.255.0` |
| Lease duration | 8 days |
| Exclusion range | `192.168.1.50` – `192.168.1.60` (reserved for static assignments) |

**Reservation:**

| Setting | Value |
|---|---|
| Client | `ENPAL-SRV01` |
| Reserved IP | `192.168.1.100` |
| MAC address | `08-00-27-1a-6f-92` |

**Verification:** Ran `ipconfig /renew` on `ENPAL-SRV01`. Confirmed it received the exact reserved address from the correct DHCP server:
```
IPv4 Address:    192.168.1.100
DHCP Server:     192.168.1.20
DNS Servers:     192.168.1.10
Lease Obtained:  29 August 2026
Lease Expires:   6 September 2026
```
This confirms the DHCP server is correctly authorized, scoped, and serving addresses/reservations to real clients on the network not just installed without error.

## Troubleshooting Log

| Issue | Root Cause | Resolution |
|---|---|---|
| New VM defaulted to unattended install, failed with "Windows cannot find the Microsoft Software License Terms" | Known VirtualBox bug with the unattended-install feature on certain ISO/version combinations | Recreated the VM with unattended installation unchecked, installed manually instead |
| `FileServer03` could `ping`/`nslookup` the DC's IP directly but with "Destination host unreachable" replies from itself | Network adapter was set to NAT instead of Internal Network completely different virtual network from the DC and client | Changed Adapter 1 to Internal Network, matching the exact network name used by the DC and client |
| First domain join attempt failed: "The specified domain either does not exist or could not be contacted" | Attempted a computer rename and domain join simultaneously in one step | Retried the domain join as an isolated step |
| `Add-DhcpServerInDC` failed with Kerberos/WinRM error via the GUI wizard | Flaky WinRM/Kerberos handshake specific to the GUI post-install wizard | Ran the equivalent `Add-DhcpServerInDC` PowerShell cmdlet directly instead |
| `Add-DhcpServerInDC` failed repeatedly with Error 20070 ("Failed to initialize directory service resources"), persisting even after fixing the system clock | **Root cause 1:** VirtualBox was reverting the VM's clock away from the correct time within seconds of manually setting it, despite the host clock itself being correct. **Root cause 2:** Separately, the command was being run while logged in as the *local* `berlin-fs-01\administrator` account rather than a domain account  local accounts have no permissions in Active Directory regardless of the account name | **For the clock:** permanently disabled VirtualBox's host time sync at the hypervisor level for both VMs via `VBoxManage setextradata <VM> "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1`, run from the host machine with each VM powered off rather than continuing to manually correct the clock every session. **For the account:** logged out and back in with a proper domain account, confirmed via `whoami` showing `divine\username` instead of `berlin-fs-01\username`, after which authorization succeeded |
| `Add-DhcpServerv4ExclusionRange` failed on a second exclusion attempt | Typo used `192.168.0.60` (wrong third octet) instead of `192.168.1.60`, placing the range in a different subnet than the scope | Not re-attempted since the first exclusion range already covered lab needs; noted here as a reminder to double-check subnet octets carefully when scripting network ranges |

## Known Simplifications / Notes

- `BERLIN-FS-01` runs Windows Server 2022, matching the DC though the lab was briefly considered with Server 2025 during initial VM creation before settling on 2022 for consistency
- Lease duration left at the 8-day default rather than tuned for a specific use case, since this is a lab environment rather than a production network with device-turnover patterns to optimize against

## Skills Demonstrated

- Multi-server Active Directory infrastructure design (role separation from the Domain Controller)
- DHCP Server installation, authorization, and scope configuration
- DHCP exclusion ranges and static reservations
- End-to-end verification methodology (not just "no errors" confirmed actual client behavior)
- VirtualBox networking troubleshooting (NAT vs. Internal Network)
- Root-cause diagnosis of a recurring Kerberos/clock-sync issue, resolved permanently at the hypervisor level rather than patched repeatedly
- Local vs. domain account permission troubleshooting in Active Directory contexts

# Windows Infrastructure Lab

A second-server infrastructure build for the `divine.lab` domain, providing DHCP, DNS support, File Server, and Backup services deliberately separated from the Domain Controller to reflect real-world role separation rather than piling every service onto one box.

## Environment

| Component | Details |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Domain Controller (existing) | Windows Server 2022 (`Berlin-DC-01`) — AD DS + DNS |
| Infrastructure Server (new) | Windows Server 2022 (`BERLIN-FS-01`, VM name `FileServer03`) DHCP, File Server, Backup |
| Domain | `divine.lab` |
| Network | Internal Network (isolated lab, static IPs) |
| BERLIN-FS-01 static IP | `192.168.1.20` |

**Design rationale:** DNS remains on the Domain Controller since it's tightly coupled with AD. DHCP, File Server, and Backup roles live on a separate member server reflecting the real-world practice of keeping Domain Controllers lean and dedicated to identity services, rather than overloading them with unrelated infrastructure roles.

## DHCP Server

**Role installation:** Installed via Server Manager → Add Roles and Features → DHCP Server, then authorized in Active Directory via PowerShell:
```powershell
Add-DhcpServerInDC -DnsName "BERLIN-FS-01.divine.lab" -IPAddress 192.168.1.20
```

**Scope configuration:**

| Setting | Value |
|---|---|
| Scope name | `Berlin_Clients_Scope` |
| Range | `192.168.1.50` – `192.168.1.150` |
| Subnet mask | `255.255.255.0` |
| Lease duration | 8 days |
| Exclusion range | `192.168.1.50` – `192.168.1.60` (reserved for static assignments) |

**Reservation:**

| Setting | Value |
|---|---|
| Client | `ENPAL-SRV01` |
| Reserved IP | `192.168.1.100` |
| MAC address | `08-00-27-1a-6f-92` |

**Verification:** Ran `ipconfig /renew` on `ENPAL-SRV01`. Confirmed it received the exact reserved address from the correct DHCP server:
```
IPv4 Address:    192.168.1.100
DHCP Server:     192.168.1.20
DNS Servers:     192.168.1.10
Lease Obtained:  29 August 2026
Lease Expires:   6 September 2026
```
This confirms the DHCP server is correctly authorized, scoped, and serving addresses/reservations to real clients on the network not just installed without error.

## File Server (FSRM Quotas)

**Roles installed:** File and Storage Services, File Server Resource Manager (FSRM).

**Department share structure:**

| Folder | Quota Limit | Type |
|---|---|---|
| `C:\Shares\IT` | 5 GB | Hard |
| `C:\Shares\Finance` | 2 GB | Hard |

**Why hard quotas:** a hard quota actively blocks writes once the limit is reached, unlike a soft quota which only sends a warning while still allowing the limit to be exceeded. For genuine storage control preventing one department from consuming the whole disk — hard quotas are the correct choice.

**Verification:** attempted to create a file larger than the IT folder's 5GB limit using `fsutil file createnew testfile.bin 5500000000`. The operation was rejected with `Error: There is not enough space on the disk`, confirming the quota is actively enforced at the file-system level, not just displayed as a configured value in the console.

## Windows Server Backup

**Feature installed:** Windows Server Backup, targeting the department share data (`C:\Shares`).

**Backup configuration:** rather than using the GUI scheduling wizard which requires dedicating an entire disk exclusively to scheduled backups — the recurring schedule was configured via `wbadmin`, which supports scheduling backups directly to a network share target:
```
wbadmin enable backup -addtarget:\\BERLIN-FS-01\LocalBackups -schedule:21:00 -include:C:\Shares -quiet
```

**Verification steps:**
1. Ran an initial manual backup (`wbadmin start backup ...`) and confirmed success via `Get-WBSummary` (`LastBackupResultHR: 0`)
2. Enabled the recurring schedule and confirmed via `Get-WBSummary` that `NextBackupTime` populated correctly, proving the schedule was genuinely registered rather than merely attempted
3. **Performed a full restore test** the step that actually proves a backup is usable, not just "completed without error":
   - Created a test file inside `C:\Shares\IT`, backed it up
   - Deleted the file to simulate real data loss
   - Used the Windows Server Backup Recovery wizard to restore the specific deleted file to its original location
   - Confirmed the restored file was present with its original content intact via `Get-Content`

This restore test is the most important part of the Backup deliverable an untested backup is only a hope, not a verified safety net.

## Troubleshooting Log

| Issue | Root Cause | Resolution |
|---|---|---|
| New VM defaulted to unattended install, failed with "Windows cannot find the Microsoft Software License Terms" | Known VirtualBox bug with the unattended-install feature on certain ISO/version combinations | Recreated the VM with unattended installation unchecked, installed manually instead |
| `FileServer03` could `ping`/`nslookup` the DC's IP directly but with "Destination host unreachable" replies from itself | Network adapter was set to NAT instead of Internal Network — completely different virtual network from the DC and client | Changed Adapter 1 to Internal Network, matching the exact network name used by the DC and client |
| First domain join attempt failed: "The specified domain either does not exist or could not be contacted" | Attempted a computer rename and domain join simultaneously in one step | Retried the domain join as an isolated step |
| `Add-DhcpServerInDC` failed with Kerberos/WinRM error via the GUI wizard | Flaky WinRM/Kerberos handshake specific to the GUI post-install wizard | Ran the equivalent `Add-DhcpServerInDC` PowerShell cmdlet directly instead |
| `Add-DhcpServerInDC` failed repeatedly with Error 20070 ("Failed to initialize directory service resources"), persisting even after fixing the system clock | **Root cause 1:** VirtualBox was reverting the VM's clock away from the correct time within seconds of manually setting it, despite the host clock itself being correct. **Root cause 2:** Separately, the command was being run while logged in as the *local* `berlin-fs-01\administrator` account rather than a domain account — local accounts have no permissions in Active Directory regardless of the account name | **For the clock:** permanently disabled VirtualBox's host time sync at the hypervisor level for both VMs via `VBoxManage setextradata <VM> "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1`, run from the host machine with each VM powered off — rather than continuing to manually correct the clock every session. **For the account:** logged out and back in with a proper domain account, confirmed via `whoami` showing `divine\username` instead of `berlin-fs-01\username`, after which authorization succeeded |
| `Add-DhcpServerv4ExclusionRange` failed on a second exclusion attempt | Typo — used `192.168.0.60` (wrong third octet) instead of `192.168.1.60`, placing the range in a different subnet than the scope | Not re-attempted since the first exclusion range already covered lab needs; noted here as a reminder to double-check subnet octets carefully when scripting network ranges |
| Restore wizard brought back the wrong file after a simulated deletion | During the recovery wizard's file browser, an existing file (`project_notes.txt`) was selected instead of the intended deleted test file, which Windows Server Backup restored as a timestamped copy rather than flagging an error | Deleted the incorrect result, re-ran the Recovery wizard, and carefully selected the correct file this time by verifying its exact name in the backup catalog before confirming the restore |

## Known Simplifications / Notes

- `BERLIN-FS-01` runs Windows Server 2022, matching the DC — though the lab was briefly considered with Server 2025 during initial VM creation before settling on 2022 for consistency
- Lease duration left at the 8-day default rather than tuned for a specific use case, since this is a lab environment rather than a production network with device-turnover patterns to optimize against

## Skills Demonstrated

- Multi-server Active Directory infrastructure design (role separation from the Domain Controller)
- DHCP Server installation, authorization, and scope configuration
- DHCP exclusion ranges and static reservations
- End-to-end verification methodology (not just "no errors" confirmed actual client behavior)
- VirtualBox networking troubleshooting (NAT vs. Internal Network)
- Root-cause diagnosis of a recurring Kerberos/clock-sync issue, resolved permanently at the hypervisor level rather than patched repeatedly
- Local vs. domain account permission troubleshooting in Active Directory contexts
- File Server Resource Manager (FSRM) quota configuration and hard-limit enforcement testing
- Windows Server Backup: scheduled job configuration via command line (`wbadmin`) as an alternative to GUI-only workflows
- Full backup/restore lifecycle verification treating "backup completed" and "data is actually recoverable" as two separate, both-necessary proofs

---

*All four core deliverables for this project DHCP, DNS integration, File Server with quotas, and Windows Server Backup with a verified restore are complete as documented above.*

