# Active Directory & Windows Security Lab
Enterprise-Grade Windows Domain Deployment & Security Hardening  
Author: ZD Homelab  

---

## Overview
This project demonstrates the deployment and security hardening of a Windows domain environment inside the Zeroday homelab. The lab replicates a small enterprise environment using:

- Windows Server 2025 (Domain Controller)
- Windows 11 Pro (Domain-joined workstation)
- Cisco networking equipment
- Proxmox VE virtualization
- Pi-hole DNS filtering

The goal is to build a functional, secure, and fully monitored Windows domain while implementing foundational identity, authentication, and workstation security controls.

---

## Objectives
- Deploy Windows Server 2025 as an enterprise-grade Active Directory Domain Controller
- Configure AD DS with secure DNS integration using Pi-hole
- Build a structured OU hierarchy for users, computers, and security groups
- Implement a domain-wide Group Policy security baseline
- Join Windows 11 workstation to the domain and validate trust
- Enforce password, lockout, and audit policies aligned with security standards
- Enable Advanced Audit Policy for authentication and policy change monitoring
- Validate GPO application using RSoP and gpresult
- Generate and analyze failed authentication events (4625, 4776)

---

## Key Outcomes
- Fully functional Windows Server 2025 domain controller with AD DS, Kerberos, and secure DNS
- Domain-wide security baseline GPO enforcing password, lockout, audit, and workstation hardening
- Advanced Audit Policy enabled and validated (4624, 4625, 4720, 4732, 4738, 4719)
- Hardened Windows 11 workstation (Firewall, Defender, SmartScreen, least privilege)
- Pi-hole DNS filtering integrated with conditional forwarding to AD DNS
- Troubleshooting documented for DNS, GPO, replication, audit gaps, and workstation permissions
- Verified domain trust, GPO inheritance, and workstation policy enforcement

---

## Lab Architecture

### Proxmox VE Host (pve01)
pve01 (Proxmox VE)
│
├── SRV-DC01 (Windows Server 2025)
│   ├── Active Directory Domain Services
│   ├── DNS (zeroday.local)
│   └── Group Policy Security Baseline
│
└── WC01 (Windows 11 Pro)
└── Domain-joined workstation

### Network LayoutCisco Router
│
└── Cisco Switch
├── Proxmox VE Host
└── Pi-hole DNS Server
├── DNS Filtering
└── Conditional Forwarding → SRV-DC01

---

## Active Directory Structure

### Organizational Units

---

## Active Directory Structure

### Organizational Units
zeroday.local
│
├── Lab-Users
├── Lab-Computers
└── Lab-Groups

### User Accounts
| Display Name    | Username  | Role           | OU        |
|-----------------|-----------|----------------|-----------|
| Mike Dt         | mike.dt   | Standard User  | Lab-Users |
| Bob Jeen        | bob.jeen  | Standard User  | Lab-Users |
| Admin ZD        | adm.zd    | Domain Admin   | Lab-Users |
| Lab Technician  | lab.tech  | Power User     | Lab-Users |
| Ali Jan         | ali.jan   | Standard User  | Lab-Users |

### Security Groups
| Group                 | Members                 |
|-----------------------|-------------------------|
| GG-IT-Admin           | Admin ZD                |
| GG-Helpdesk           | Lab Technician          |
| GG-Standard-Users     | Mike Dt, Bob Jeen       |
| GG-Workstation-Admins | Admin ZD                |
| GG-GPO-Workstations   | Mike Dt, Bob Jeen       |
| GG-Lab-Admin          | Admin ZD                |

---

## Group Policy Security Baseline

### Password Policy
- Minimum length: 14 characters
- Password history: 10 remembered
- Complexity: Disabled
- Expiry: None
- Reversible encryption: Disabled

### Account Lockout Policy
- Threshold: 10 invalid attempts
- Duration: 30 minutes
- Reset counter: 30 minutes

### Advanced Audit Policy
Enabled for:
- Account Logon
- Account Management
- Logon/Logoff
- Policy Change
- Security Group Management
- User Account Management

---

## Validation & Testing

### Password & Lockout Policy
- Verified 14-character minimum
- Confirmed password history enforcement
- Tested lockout at 10 failed attempts
- Verified 30-minute lockout duration

### Audit Policy Validation
Events confirmed on SRV-DC01 and WC01:
- 4624 – Successful logon
- 4625 – Failed logon
- 4720 – User account created
- 4732 – Added to security group
- 4738 – User account changed
- 4719 – Audit policy changed

### Failed Logon Test
- Wrong password attempt for bob.jeen
- Event 4625 logged (NTLM, logon type 3)
- Substatus 0xC000006A → valid account, bad password
- Matching 4776 credential validation failure

### Workstation Hardening Validation
- Firewall enabled
- Defender real-time protection active
- SmartScreen enabled
- Standard users have no local admin rights
- Admin ZD has admin rights via GG-Workstation-Admins

---

## Challenges & Troubleshooting
- DNS misconfiguration prevented domain join → fixed by pointing WC01 to SRV-DC01
- GPO not applying → resolved with gpupdate /force and correct OU linking
- Missing audit events → fixed by enabling Advanced Audit Policy
- Standard users had unintended admin rights → corrected via security groups
- Pi-hole required conditional forwarding for AD DNS functionality

---

## Lessons Learned

### Active Directory Design
- OU structure is critical for clean GPO targeting
- Separating standard users, admin users, and service accounts prevents privilege creep

### Group Policy Management
- Domain-level policies override local settings
- Advanced Audit Policy provides more detailed and useful logging than legacy audit settings

### Security Best Practices
- Least privilege should be enforced through security groups, not local configuration
- Password and lockout policies significantly strengthen domain security with minimal user impact

### Troubleshooting Workflow
- gpresult /r and Event Viewer are essential tools for diagnosing GPO issues
- DNS and time synchronization are foundational; Active Directory breaks quickly without them

### Documentation & Validation
- Screenshots and structured validation steps make the environment reproducible and portfolio-ready

---

## Future Enhancements

### Local Administrator Password Solution (LAPS)
Deploying Windows LAPS ensures unique, automatically rotated local admin passwords across all domain-joined systems, reducing lateral movement risk.

### Sysmon + Wazuh Integration
Adding Sysmon for detailed endpoint telemetry and forwarding logs to Wazuh SIEM enhances threat detection, process monitoring, and incident investigation.

### Network Segmentation
Implementing VLANs on the Cisco switch isolates domain controllers, workstations, and management interfaces, reducing broadcast traffic and limiting lateral movement opportunities.

### Kerberos-Only Authentication
Disabling NTLM and enforcing Kerberos-only authentication strengthens identity security and prevents NTLM relay attacks. This requires validating all services for Kerberos compatibility.

### BitLocker + TPM Enforcement
Encrypting workstation drives with BitLocker protects data at rest and mitigates physical access attacks.

### AppLocker or WDAC
Application control policies restrict unauthorized software execution, reducing malware risk and enforcing a trusted application baseline.

---

## Technologies Used

### Operating Systems & Directory Services
- Windows Server 2025 (Domain Controller)
- Windows 11 (WC01 workstation)
- Active Directory Domain Services (AD DS)

### Group Policy & Security
- Group Policy Management Console (GPMC)
- Advanced Audit Policy
- Windows Defender
- Windows Firewall
- SmartScreen

### Networking & Infrastructure
- Pi-hole DNS
- Cisco Networking Equipment
- Proxmox VE Virtualization

### Tools & Utilities
- Event Viewer
- gpresult /r
- gpupdate /force
- Active Directory Users and Computers (ADUC)
- PowerShell

