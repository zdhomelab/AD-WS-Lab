# Active Directory & Windows Security Lab

![Windows Server](https://img.shields.io/badge/Windows%20Server-2025-0078D4?logo=windows&logoColor=white)
![Windows 11](https://img.shields.io/badge/Windows-11%20Pro-0078D4?logo=windows11&logoColor=white)
![Proxmox](https://img.shields.io/badge/Proxmox-VE-E57000?logo=proxmox&logoColor=white)
![Pi-hole](https://img.shields.io/badge/Pi--hole-DNS%20Filtering-96060C?logo=pihole&logoColor=white)
![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?logo=cisco&logoColor=white)

Deployment and security hardening of a Windows domain environment inside the **Zeroday homelab**, built on Cisco physical networking gear and a Proxmox VE hypervisor. The environment includes a Windows Server 2025 domain controller, Active Directory Domain Services (AD DS), Pi-hole DNS filtering, and a domain-joined Windows 11 workstation.

The goal was to build a functional Windows domain while implementing foundational security controls across identity management, password and account lockout policies, Windows auditing, DNS-level filtering, and authentication monitoring. The setup mirrors the core components of a small enterprise environment and provides hands-on experience with Windows security, Group Policy, and domain administration.

---

##  Table of Contents

- [Objectives](#-objectives)
- [Key Outcomes](#-key-outcomes)
- [System Components](#-system-components)
- [Architecture](#-architecture)
- [Active Directory Structure](#-active-directory-structure)
- [Group Policy Security Baseline](#-group-policy-security-baseline)
- [Validation & Testing](#-validation--testing)
- [Screenshots](#-screenshots)
- [Challenges & Troubleshooting](#-challenges--troubleshooting)
- [Lessons Learned](#-lessons-learned)
- [Future Enhancements](#-future-enhancements)
- [Technologies Used](#-technologies-used)

---

##  Objectives

- Deploy **Windows Server 2025** as an enterprise-grade **Active Directory Domain Controller**
- Configure **AD DS** and integrate DNS with **Pi-hole** for upstream filtering and name resolution
- Create a structured **OU hierarchy**, domain users, and security groups following best-practice design
- Implement and enforce a **Group Policy security baseline** (workstation hardening and identity protections)
- Join a **Windows 11 client** to the domain and validate domain trust operations
- Configure **password length and history**, **expiry (no forced rotation)**, and **account lockout** policies aligned with security standards
- Enable and tune **Windows security auditing** (logon events, policy changes, directory service access)
- Validate applied policies using **Resultant Set of Policy (RSoP)** and Group Policy Modeling
- Generate and analyze a **failed authentication event (Event ID 4625)** to confirm auditing and monitoring work

---

##  Key Outcomes

- Built a Windows Server 2025 domain controller with AD DS, Kerberos authentication, and secure DNS integration
- Implemented a domain-wide security baseline GPO enforcing password, lockout, audit, and workstation hardening policies
- Enabled full Advanced Audit Policy logging and validated events **4624, 4625, 4720, 4732, 4738, and 4719**
- Hardened the Windows 11 workstation via Group Policy (Firewall, Defender, SmartScreen, least privilege)
- Integrated Pi-hole DNS filtering with conditional forwarding to Active Directory DNS
- Documented troubleshooting for DNS, GPO processing, replication, audit policy gaps, and workstation permissions
- Validated domain trust, GPO inheritance, and policy enforcement using `gpresult`, RSoP, and Event Viewer

---

##  System Components

### Table 1. Lab components

| Component | Platform | OS / Software | Role | Key Configuration |
|---|---|---|---|---|
| **Cisco router & switch** | Physical network equipment | Cisco IOS | Lab network connectivity | DHCP, basic routing, switchport security; no VLAN segmentation |
| **pve01** | Physical host | Proxmox VE (KVM/QEMU) | Hypervisor for lab VMs | Hosts SRV-DC01 and WC01 |
| **SRV-DC01** (`192.168.10.10`) | Virtual machine | Windows Server 2025 | Domain controller; DNS for `zeroday.local` | AD DS, Group Policy, Kerberos, security auditing, NTP time sync |
| **Pi-hole** | Linux server | Pi-hole (latest) | DNS filtering and upstream resolver | Blocks ad/malicious domains; conditional forwarding of `zeroday.local` to SRV-DC01 |
| **WC01** | Virtual machine | Windows 11 Pro | Domain-joined workstation | Joined to `zeroday.local`; GPO applied; auditing enabled; used to test policies and auth events |

### Table 2. Security controls and validation methods

| Control / Method | Scope | Configuration | Validation |
|---|---|---|---|
| **Password policy** | Domain (GPO-Security Baseline) | Min length 14; history 10; complexity disabled; no forced expiry | GPMC settings view; password change tests |
| **Account lockout policy** | Domain (GPO-Security Baseline) | Threshold 10 invalid attempts; duration 30 min; counter reset after 30 min | Repeated failed logons; lockout behavior confirmed |
| **Advanced audit policy** | Domain (GPO-Security Baseline) | Logon/Logoff, account logon, account management, policy change, security group mgmt, user account mgmt | Event Viewer on SRV-DC01 and WC01: 4624, 4625, 4720, 4732, 4738, 4719 |
| **Workstation hardening** | WC01 | Windows Firewall, Defender real-time protection, SmartScreen enabled; standard users without local admin | `gpresult /r` and RSoP on WC01 |
| **Failed logon test** | WC01 → SRV-DC01 | Deliberate wrong-password network logon as `bob.jeen` | Events 4625 (logon type 3, NTLM) and 4776 logged on SRV-DC01; source WC01 |
| **Policy verification** | WC01, SRV-DC01 | `gpresult /r`; RSoP MMC | Confirms GPO-Security Baseline and default domain policies are applied and inherited |

---

##  Architecture

```text
Cisco Router
└── Cisco Switch
    ├── Proxmox VE Host (pve01)
    │   └── Proxmox VE Hypervisor (KVM/QEMU)
    │       ├── SRV-DC01 (VM)
    │       │   ├── Windows Server 2025
    │       │   ├── Active Directory Domain Services (AD DS)
    │       │   ├── Domain DNS (zeroday.local)
    │       │   └── Group Policy Security Baseline
    │       │
    │       └── WC01 (VM)
    │           ├── Windows 11 Pro
    │           └── Joined to zeroday.local domain
    │
    └── Pi-hole DNS Server
        ├── DNS filtering
        └── Forwards AD DNS to SRV-DC01
```

---

##  Active Directory Structure

The domain is organized into dedicated Organizational Units (OUs) to separate users, computers, and security groups, giving clear administrative boundaries and simpler Group Policy management.

### Organizational Units

| OU | Purpose |
|---|---|
| **Lab-Users** | All standard and administrative user accounts in `zeroday.local` |
| **Lab-Computers** | Domain-joined workstation and server objects, including the Windows 11 VM |
| **Lab-Groups** | Security groups used for access control, permissions, and GPO targeting |

### User Accounts

| Display Name | Username (sAMAccountName) | Role | OU Location | Notes |
|---|---|---|---|---|
| Mike Dt | `mike.dt` | Standard User | `zeroday.local/Lab-Users` | General workstation user |
| Bob Jeen | `bob.jeen` | Standard User | `zeroday.local/Lab-Users` | Used for GPO testing |
| Admin ZD | `admin.zd` | Domain Admin | `zeroday.local/Lab-Users` | Primary administrative account |
| Lab Technician | `lab.tech` | Power User | `zeroday.local/Lab-Users` | Elevated permissions for lab operations |
| Ali Jan | `ali.jan` | Standard User | `zeroday.local/Lab-Users` | General workstation user |

### Security Groups and Members

| Group | Members |
|---|---|
| **GG-IT-Admin** | admin.zd |
| **GG-Helpdesk** | lab.tech |
| **GG-Standard-Users** | mike.dt, bob.jeen |
| **GG-Workstation-Admins** | admin.zd |
| **GG-GPO-Workstations** | mike.dt, bob.jeen |
| **GG-Lab-Admin** | admin.zd |

---

##  Group Policy Security Baseline

A domain-level GPO named **GPO – Security Baseline** enforces foundational security controls across the `zeroday.local` environment.

<table>
<tr>
<td valign="top" width="50%">

**Password Policy**

| Setting | Value |
|---|---|
| Password history | 10 passwords |
| Maximum password age | 0 days |
| Minimum password age | 0 days |
| Minimum length | 14 characters |
| Complexity | Disabled |
| Reversible encryption | Disabled |

</td>
<td valign="top" width="50%">

**Account Lockout Policy**

| Setting | Value |
|---|---|
| Lockout threshold | 10 invalid attempts |
| Lockout duration | 30 minutes |
| Reset counter after | 30 minutes |

</td>
</tr>
</table>

**Advanced Audit Policy** covers: Account Logon · Account Management · Logon/Logoff · Policy Change · Authentication Activity · Security Group Management · User Account Management

**Applied to:** all users and computers in `zeroday.local`, including WC01, SRV-DC01, all five user accounts, and all six security groups.

---

##  Validation & Testing

### Password & Account Policy
- Verified the **14-character** minimum length during user password changes
- Tested password history enforcement (10 previous passwords remembered)
- Ran repeated failed logins to confirm the lockout threshold triggers at **10 attempts**, the account stays locked for **30 minutes**, and the counter resets after **30 minutes**

### Advanced Audit Policy
Logs reviewed on **SRV-DC01** and **WC01** confirmed recording of account logon (success and failure), account management, logon/logoff, policy change, security group management, and user account management events.

### Event Viewer Checks

| Event ID | Meaning | Result |
|---|---|---|
| **4624** | Successful logon | Recorded on SRV-DC01 for a successful network logon (type 3, NTLM) of `bob.jeen` from WC01 (`192.168.10.69`), matching the earlier failed attempt |
| **4625** | Failed logon | Failed network logon (type 3, NTLM) for `bob.jeen`. Status `0xC000006D` / sub-status `0xC000006A` = valid account, wrong password. Source: WC01 (`192.168.10.69`); a matching **4776** was logged at the same time |
| **4720** | User account created | `ali.jan` created by the Administrator account, confirming User Account Management auditing |
| **4732** | User added to security group | Verified |
| **4738** | User account changed | Verified |
| **4719** | System audit policy changed | Verified |

This confirms failed authentication attempts are audited and traceable to a source host and IP address.

### Workstation Policy (WC01)
- Confirmed the baseline was received via `gpresult /r`
- Verified **Firewall** enabled, **Defender real-time protection** active, **SmartScreen** enabled, and **audit policy** applied
- Confirmed standard users (Mike Dt, Bob Jeen) have **no** local admin rights
- Confirmed Admin ZD has local admin rights via **GG-Workstation-Admins**

### Domain Controller
- Confirmed DC-specific audit policies applied
- Verified secure DNS updates

---

## 📸 Screenshots

### Event Viewer Validation

| Event 4624: Successful logon | Event 4625: Failed logon |
|:---:|:---:|
| <img src="github-lab-readme/screenshots/01-event-4624.png" alt="Event 4624"> | <img src="github-lab-readme/screenshots/02-event-4625.png" alt="Event 4625"> |

**Event 4720: User account created**

| <img src="github-lab-readme/screenshots/03-event-4720.png" alt="Event 4720" width="75%"> | <img src="github-lab-redme/screenshots/03-event-4720.png" alt="Event 4720" width="75%"> |

### Domain Controller Build

**Server Manager: SRV-DC01 local server properties**

<img src="github-lab-readme/screenshots/04-server-manager.png" alt="Server Manager" width="75%">

**IP configuration (`ipconfig`)**

<img src="github-lab-readme/screenshots/05-ip-configuration.png" alt="IP configuration" width="75%">

**AD DS and DNS role installation**

<img src="github-lab-readme/screenshots/06-adds-installation.png" alt="AD DS installation" width="75%">

**AD DS Configuration Wizard: prerequisites passed**

<img src="github-lab-readme/screenshots/07-ad-prerequisites.png" alt="AD prerequisites" width="75%">

**Domain verification: `zeroday.local` with SRV-DC01 as infrastructure master**

<img src="github-lab-readme/screenshots/08-domain-verification.png" alt="Domain verification" width="75%">

**DNS resolution: `nslookup` resolves `srv-dc01.zeroday.local` to `192.168.10.10`**

<img src="github-lab-readme/screenshots/09-dns-resolution.png" alt="DNS resolution" width="75%">

### Active Directory Structure

**Organizational units: Lab-Users, Lab-Computers, Lab-Groups**

<img src="github-lab-readme/screenshots/10-organizational-units.png" alt="Organizational units" width="75%">

**Lab-Users OU with the five test accounts**

<img src="github-lab-readme/screenshots/11-test-users.png" alt="Test users" width="75%">

**Lab-Groups OU with the six security groups**

<img src="github-lab-readme/screenshots/12-security-groups.png" alt="Security groups" width="75%">

### Security Configuration

**GPO-Security Baseline linked at the domain level**

<img src="github-lab-readme/screenshots/13-gpo-security-baseline.png" alt="GPO Security Baseline" width="75%">

| Password policy | Account lockout policy |
|:---:|:---:|
| <img src="github-lab-readme/screenshots/14-password-policy.png" alt="Password policy"> | <img src="screenshots/15-lockout-policy.png" alt="Lockout policy"> |

**Domain controller GPO verification (SRV-DC01):** Default Domain Controllers Policy, Default Domain Policy, and GPO-Security Baseline applied successfully.

<img src="github-lab-readme/screenshots/16-dc-gpo-verification.png" alt="DC GPO verification" width="75%">

**Workstation GPO verification (WC01):** Default Domain Policy and GPO-Security Baseline applied from SRV-DC01.

<img src="github-lab-readme/screenshots/17-wc01-gpo-verification.png" alt="WC01 GPO verification" width="75%">

### DNS & Pi-hole Integration

Pi-hole forwards queries for `zeroday.local` to the domain controller using **Conditional Forwarding**, so AD DNS stays authoritative for domain queries while Pi-hole filters external DNS traffic.

| Setting | Value |
|---|---|
| Network range | `192.168.10.0/24` |
| Server IP | `192.168.10.10` (Domain Controller) |
| Domain | `zeroday.local` |

<img src="github-lab-readme/screenshots/18-pihole-conditional-forwarding.png" alt="Pi-hole conditional forwarding" width="75%">

**DNS Manager:** all required AD SRV records present (LDAP, Kerberos, Global Catalog, DC locator) with no errors or stale entries.

<img src="github-lab-readme/screenshots/19-dns-manager.png" alt="DNS Manager" width="75%">

**`dcdiag /test:dns`:** DNS, ForestDnsZones, DomainDnsZones, Schema, Configuration, and enterprise DNS tests all passed.

<img src="github-lab-readme/screenshots/20-dcdiag-dns.png" alt="dcdiag DNS test" width="75%">

### Replication Health

Replication tests were run even in a single-DC environment to validate the AD replication engine across Schema, Configuration, Domain, ForestDnsZones, and DomainDnsZones partitions. `repadmin /replsummary` reported no errors.

<img src="github-lab-readme/screenshots/21-replication-health.png" alt="Replication health" width="75%">

### Domain-Joined Client

**WC01 joined to `zeroday.local` (Windows 11 Pro, System > About)**

<img src="github-lab-readme/screenshots/22-domain-joined-client.png" alt="Domain-joined client" width="75%">

---

##  Challenges & Troubleshooting

| Area | Issue | Resolution |
|---|---|---|
| **AD configuration** | Domain join failed because DNS was not pointing to the domain controller | Corrected DNS settings on WC01 |
| **AD configuration** | New users/groups didn't appear immediately in the console | Refreshing the view displayed them |
| **Group Policy** | GPO settings didn't apply at first | `gpupdate /force` and verified the OU link |
| **Group Policy** | Password and lockout policies only worked at the domain level; workstation-level GPOs were ignored | Applied the policies at the domain level |
| **Audit logging** | Some events were missing because default audit settings were still active | Enabled Advanced Audit Policy |
| **Audit logging** | Needed proper filtering to find 4738, 4624, 4625, 4720, 4732, 4719 | Applied Event Viewer filters |
| **Workstation permissions** | Standard users had unintended local admin rights from the initial client build | Enforced **GG-Workstation-Admins** |
| **Workstation permissions** | Admin ZD lacked local admin rights on WC01 | Added GG-Workstation-Admins to the local Administrators group via policy |
| **Networking** | Pi-hole couldn't resolve AD names properly | Added conditional forwarding to the domain controller |

---

##  Lessons Learned

- **AD design:** a proper OU structure is essential for clean GPO targeting and scalability; separating standard, admin, and service accounts prevents privilege creep
- **Group Policy:** domain-level policies override local settings, so understanding GPO precedence is critical; Advanced Audit Policy gives far more useful logging than legacy audit settings
- **Security:** enforce least privilege through security groups, not local configuration; password and lockout policies strengthen security with minimal user impact
- **Troubleshooting:** `gpresult /r` and Event Viewer are essential for diagnosing GPO issues; DNS and time sync are foundational, and AD breaks quickly without them
- **Documentation:** screenshots and structured validation steps make the environment reproducible and portfolio-ready

---

##  Future Enhancements

- **Windows LAPS:** unique, automatically rotated local admin passwords to reduce lateral movement risk
- **Sysmon + Wazuh:** endpoint telemetry forwarded to a SIEM for process, network, and registry visibility
- **Network segmentation:** VLANs on the Cisco switch to isolate domain controllers, workstations, and management interfaces
- **Kerberos-only authentication:** disable NTLM to prevent relay attacks (after validating service compatibility)
- **BitLocker + TPM enforcement:** encrypt workstation drives and protect keys with TPM
- **AppLocker / WDAC:** application control to restrict unauthorized software and enforce a trusted baseline

---

##  Technologies Used

| Category | Technologies |
|---|---|
| **Operating systems & directory** | Windows Server 2025, Windows 11, Active Directory Domain Services |
| **Group Policy & security** | GPMC, Advanced Audit Policy, Windows Defender, Windows Firewall, SmartScreen |
| **Networking & infrastructure** | Pi-hole DNS, Cisco networking equipment, Proxmox VE |
| **Tools & utilities** | DNS Manager, `dcdiag`, `repadmin`, `nslookup`, Active Directory Administrative Center, Server Manager, Event Viewer, `gpresult /r`, `gpupdate /force`, ADUC, PowerShell |

---

<p align="center"><i>Built in the Zeroday homelab </i></p>
<p align="center">
  <a href="https://www.linkedin.com/in/mohammad-h-masir">
    <img src="https://img.shields.io/badge/LinkedIn-Mohammad%20Masir-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn">
  </a>
  <a href="https://github.com/zdhomelab">
    <img src="https://img.shields.io/badge/GitHub-zdhomelab-181717?style=for-the-badge&logo=github&logoColor=white" alt="GitHub">
  </a>
</p>
