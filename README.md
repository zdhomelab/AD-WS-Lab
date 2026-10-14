<a href="https://www.linkedin.com/in/mohammad-h-masir">
  <img src="https://img.shields.io/badge/LinkedIn-Mohammad%20H.%20Masir-lightblue" />
</a>

---

# ZeroDay Homelab – Active Directory & Security Baseline

A fully configured Windows Active Directory environment deployed under the **zeroday.local** domain.  
This project demonstrates domain design, user/group management, Group Policy security baselines, auditing, and validation.

---

## 1. Active Directory Structure

The domain environment was organized using dedicated organizational units (OUs) to separate users, computers, and security groups.

### Organizational Units
- Lab-Users  
- Lab-Computers  
- Lab-Groups  

### OU Layout
- zeroday.local  
  - Lab-Users  
    - Standard  
    - Admin  
  - Lab-Computers  
    - Workstations  
    - Servers  
  - Lab-Groups  
    - Security Groups  
    - GPO Targeting Groups
---

## 2. User Accounts

| Display Name | Username (sAMAccountName) | Role | OU Location | Notes |
| --- | --- | --- | --- | --- |
| John Doe | john.doe | Standard User | `zeroday.local/Lab-Users/Standard` | General workstation user |
| Sarah Miller | sarah.miller | Standard User | `zeroday.local/Lab-Users/Standard` | Used for GPO testing |
| Admin ZD | adm.zd | Domain Admin | `zeroday.local/Lab-Users/Admin` | Primary administrative account |
| Lab Technician | lab.tech | Power User | `zeroday.local/Lab-Users/Admin` | Elevated permissions for lab operations |

---

## 3. Security Groups

### Group Memberships

| Group | Members |
|-------|---------|
| GG-IT-Admins | Admin ZD |
| GG-Helpdesk | Lab Technician |
| GG-VPN-Users | John Doe, Sarah Miller |
| GG-Standard-Users | John Doe, Sarah Miller |
| GG-Workstation-Admins | Admin ZD |
| GG-GPO-Workstations | John Doe, Sarah Miller |
| GG-Lab-Admins | Admin ZD |

### Group Structure
zeroday.local
└── Lab-Groups
├── GG-IT-Admins
├── GG-Helpdesk
├── GG-VPN-Users
├── GG-Standard-Users
├── GG-Workstation-Admins
├── GG-GPO-Workstations
└── GG-Lab-Admins

---

## 4. Group Policy Security Baseline

A domain-level Group Policy Object (GPO) named **GPO – Security Baseline** was created to establish baseline security controls across the Windows environment.

### Password Policy
- Password history: 5 passwords remembered  
- Maximum password age: 90 days  
- Minimum password age: 1 day  
- Minimum password length: 12 characters  
- Password complexity: Enabled  
- Reversible password encryption: Disabled  

### Account Lockout Policy
- Account lockout threshold: 5 invalid attempts  
- Account lockout duration: 15 minutes  
- Reset account lockout counter after: 15 minutes  

### Advanced Audit Policy
Auditing was configured for:
- Account Logon  
- Account Management  
- Logon/Logoff  
- Policy Change  
- Authentication Activity  
- Security Group Management  
- User Account Management  

---

## 5. Validation & Testing

### Password & Lockout Validation
- Verified password complexity and minimum length  
- Confirmed password history enforcement  
- Tested lockout after 5 failed attempts  
- Verified lockout duration and reset timer  

### Audit Policy Validation
Event Viewer logs confirmed:
- 4624 – Successful logon  
- 4625 – Failed logon  
- 4720 – User account created  
- 4732 – Added to security group  
- 4738 – User account changed  
- 4719 – Audit policy changed  

### Workstation Validation (CLIENT01)
- `gpresult /r` confirmed GPO application  
- Firewall enabled  
- Defender real-time protection active  
- SmartScreen enabled  
- Standard users not in local Administrators  
- Admin ZD present in local Administrators  

### Domain Controller Validation
- DC audit policies applied  
- Secure DNS updates enabled  
- Kerberos authentication functioning correctly  

---

## 6. Visual Diagram – Screenshot Placement
GPO - Security Baseline
───────────────────────
• GPMC: GPO listed
• Password Policy
• Lockout Policy
• Audit Policy

Policy Application
───────────────────
• gpresult /r (CLIENT01)
• RSOP

Active Directory
───────────────────
• Lab-Users OU
• Lab-Groups OU
• Member Of tabs

Event Viewer
─────────────
• 4624, 4625
• 4720, 4732
• 4719

Workstation Validation
───────────────────────
• Local Administrators group
• Defender / Firewall
• SmartScreen

Domain Controller Checks
─────────────────────────
• DC audit policy
• DNS secure updates
• Kerberos logs

---

## 7. Screenshots Checklist

| Category | Screenshot Description | Purpose |
|----------|------------------------|---------|
| GPO Baseline | GPO listed in GPMC | Confirms GPO exists |
| | GPO linked to zeroday.local | Confirms correct linkage |
| | Password Policy | Shows password rules |
| | Lockout Policy | Shows lockout rules |
| | Audit Policy | Shows audit configuration |
| gpresult /r | RSOP output | Confirms GPO applied |
| Event Viewer | 4624, 4625 | Logon auditing |
| | 4720, 4732 | Account/group management |
| | 4719 | Policy change auditing |
| ADUC | Lab-Users OU | Shows user placement |
| | Lab-Groups OU | Shows group structure |
| | Member Of tabs | Confirms group membership |
| Workstation | Local Administrators | Confirms admin rights |
| | Defender/Firewall | Confirms hardening |
| Domain Controller | DNS secure updates | Confirms DNS security |
| | Kerberos logs | Confirms authentication |

---

## 9. Challenges & Troubleshooting

### Active Directory Configuration
- Initial domain join attempts failed due to incorrect DNS pointing; resolved by directing CLIENT01 to the domain controller’s IP.
- User replication delays required forcing AD replication to ensure new users and groups appeared correctly.

### Group Policy Application
- GPO settings did not apply on first attempt; resolved by running `gpupdate /force` and verifying OU linkage.
- Password and lockout policies required domain-level enforcement; workstation-level GPOs were ignored until corrected.

### Audit Policy Logging
- Some audit events were missing due to default audit settings; enabling Advanced Audit Policy resolved the issue.
- Event Viewer filtering needed refinement to properly capture 4624, 4625, 4720, 4732, and 4719 events.

### Workstation Permissions
- Standard users initially had unintended local admin rights due to default VirtualBox configuration; corrected by enforcing GG-Workstation-Admins.
- Admin ZD required manual addition to local Administrators on CLIENT01 before GPO enforcement was finalized.

### Networking & Time Sync
- Kerberos authentication failed due to clock drift; resolved by syncing time with the Raspberry Pi NTP server.
- Pi-hole DNS required conditional forwarding to the domain controller for proper AD functionality.

---

## 10. Lessons Learned

### Active Directory Design
- Proper OU structure is essential for clean GPO targeting and long-term scalability.
- Separating standard users, admins, and service accounts prevents privilege creep.

### Group Policy Management
- Domain-level policies override local settings; understanding precedence is critical.
- Advanced Audit Policy provides far more granular logging than legacy audit settings.

### Security Best Practices
- Least privilege must be enforced through security groups, not local configuration.
- Password and lockout policies significantly strengthen domain security with minimal user impact.

### Troubleshooting Workflow
- `gpresult /r` and Event Viewer are indispensable tools for diagnosing GPO issues.
- DNS and time synchronization are foundational — AD breaks quickly without them.

### Documentation & Validation
- Screenshots and structured validation steps make the environment reproducible and portfolio-ready.
- Visual diagrams help map how each component interacts within the domain.

---

## 11. Technologies Used

### Operating Systems & Directory Services
- **Windows Server 2025** (Domain Controller)
- **Windows 11** (CLIENT01 workstation)
- **Active Directory Domain Services (AD DS)**

### Group Policy & Security
- **Group Policy Management Console (GPMC)**
- **Advanced Audit Policy**
- **Windows Defender**
- **Windows Firewall**
- **SmartScreen**

### Networking & Infrastructure
- **Pi-hole DNS**
- **Raspberry Pi NTP Server**
- **Cisco Networking Equipment**
- **VirtualBox Virtualization**

### Tools & Utilities
- **Event Viewer**
- **gpresult /r**
- **gpupdate /force**
- **Active Directory Users and Computers (ADUC)**
- **PowerShell**
---



