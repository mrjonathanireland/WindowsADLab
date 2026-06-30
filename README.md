# Active Directory Lab

## Overview

A home lab simulating an enterprise Windows Server 2022 environment with Active Directory Domain Services, DNS, DHCP, organizational structure, and Group Policy enforcement. Built to demonstrate core Windows Server and sysadmin competencies in a controlled, isolated environment.

---

## Environment

| Component | Details |
|---|---|
| Domain Controller | Windows Server 2022 |
| Client Machine | Windows 11 Pro |
| Hypervisor | Proxmox VE |
| Domain | webdomain.local |
| Lab Network | Isolated internal bridge (vmbr1) |
| DC IP | 192.168.10.1 (static) |
| DHCP Range | 192.168.10.100 – 192.168.10.200 |

---

## Network Architecture

The lab runs on an isolated internal bridge (vmbr1) in Proxmox with no connection to the home network. The domain controller owns the 192.168.10.0/24 subnet and serves as the sole DNS and DHCP authority for all domain clients. This mirrors how an enterprise environment isolates domain infrastructure from general network traffic.

```
Proxmox Host
└── vmbr1 (isolated internal bridge, no physical NIC)
    ├── Windows Server 2022 — 192.168.10.1 (static)
    └── Windows 10 Pro     — 192.168.10.x (DHCP)
```

---

## What Was Built

- Promoted Windows Server 2022 to Domain Controller with AD DS
- Configured DNS with forward and reverse lookup zones for webdomain.local
- Configured DHCP scope serving the 192.168.10.x range with DNS option pointing to the DC
- Designed an OU structure reflecting a small enterprise with IT and HR departments
- Created user accounts and security groups scoped to their respective OUs
- Joined a Windows 11 Pro client to the domain
- Implemented and verified three Group Policy Objects targeting different OUs

---

## OU Structure

```
webdomain.local
└── General
    ├── IT
    │   └── ITUser (member of IT Users security group)
    └── HR
        └── HRUser (member of HR Users security group)
```

OUs are structured to reflect real departmental separation in an enterprise environment. Policies are scoped to each OU so that IT and HR users receive different permissions and restrictions based on their role, without affecting the broader domain.

---

## Group Policy Objects

| GPO | Linked To | Purpose |
|---|---|---|
| Password Policy | General OU | Enforces password complexity, length, and expiration across all domain users |
| Removable Storage Restriction | HR OU | Blocks USB and external storage devices for HR users |
| Local Admin Rights | IT OU | Grants IT users local administrator rights on domain machines |

### Password Policy
Configured under Computer Configuration and linked at the General OU level so it applies domain-wide. Enforces a baseline security standard across all users — password complexity, minimum length, and expiration are standard requirements in any enterprise environment.

### Removable Storage Restriction
Configured under User Configuration so the policy follows the HR user regardless of which machine they log into. HR handles sensitive employee and payroll data — restricting removable storage is a standard data loss prevention control and common compliance requirement.

### Local Admin Rights
Implemented via Group Policy Preferences → Local Users and Groups rather than the legacy Restricted Groups method. Uses Item Level Targeting scoped to the IT Users security group so only IT accounts receive elevation, not all users on the machine. This reflects real tiered access models where IT staff need local admin rights to perform support tasks without requiring Domain Admin credentials.

---

## Verification

**Password Policy**
- Confirmed by attempting to set a non-compliant password in ADUC — rejected as expected

**Removable Storage Restriction**
- Confirmed via Group Policy Results Wizard on the server targeting the HR user and client machine
- GPO appears in applied policies for HR user

**Local Admin Rights**
- Confirmed via `whoami /groups` in an elevated terminal on the client logged in as IT user
- BUILTIN\Administrators visible in group membership

All GPOs verified using `gpupdate /force` followed by `gpresult /r` on the client after each policy change.

---

## Troubleshooting

**Client not receiving DHCP lease**
Client had two NICs — one on vmbr1 and one on the home router bridge. The home router NIC was removed from the VM and `ipconfig /release` followed by `ipconfig /renew` resolved the lease issue. A client with two NICs on different networks will not reliably pull a lease from the correct DHCP server.

**Domain not resolving**
Client was receiving DNS from the home router instead of the DC, so webdomain.local could not be resolved. Fixed by setting DHCP Scope Option 006 (DNS Servers) to 192.168.10.1, ensuring all clients point to the DC for DNS.

**Domain join failing**
Initial join attempt used a standard domain user account. Domain join requires rights to add computer objects to the domain — resolved by using the built-in Administrator account.

**GPO not applying to IT user**
GPO was linked correctly and the user was in the correct security group. Issue was that Group Policy Preferences apply at the computer level and UAC splits the user token — a standard terminal does not show elevated group membership. Confirmed working by running `whoami /groups` in an elevated terminal, which correctly showed BUILTIN\Administrators.

---

## Skills Demonstrated

- Windows Server 2022 domain controller promotion and configuration
- Active Directory Users and Computers — OU design, user and group management
- DNS — forward and reverse lookup zone configuration
- DHCP — scope creation, lease management, and scope options
- Group Policy — GPO creation, linking, scoping, and verification
- Group Policy Preferences — Local Users and Groups with Item Level Targeting
- Proxmox — isolated virtual network design using an internal bridge
- Troubleshooting — systematic diagnosis using gpresult, ipconfig, and whoami

---

## Screenshots

See the `screenshots/` directory for:

- Server Manager dashboard with all roles active
- ADUC showing OU tree and user accounts
- DHCP scope with active leases
- DNS Manager showing forward lookup zone
- Group Policy Management showing GPOs linked to their respective OUs
- GPO editor views for each of the three policies
- Client `ipconfig /all` showing domain-issued IP and DNS pointing to 192.168.10.1
- System Properties showing client joined to webdomain.local
- `whoami /groups` output confirming BUILTIN\Administrators for IT user
