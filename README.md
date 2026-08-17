# Enterprise Active Directory Lab

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
