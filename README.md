# Enterprise Active Directory & Identity Lab

A hands-on Active Directory lab simulating enterprise system administration tasks, including identity management, Group Policy security baselines, RBAC, and troubleshooting.

This environment was extended with select cloud identity concepts using Microsoft Entra ID to reflect modern enterprise practices.

---

## Lab Scope

This project focuses on:

- Active Directory deployment and administration  
- Group Policy security configuration  
- Role-Based Access Control (RBAC)  
- PowerShell automation at scale  
- Real-world troubleshooting scenarios  
- Introductory cloud identity concepts  

---

## Environment

- Windows Server 2022 (Domain Controller)  
- Windows 10 client  
- Oracle VirtualBox  
- Internal network with DHCP/DNS  

---

## Implementation Steps

### 1. Domain Controller Deployment
- Installed AD DS role  
- Promoted server to domain controller  
- Configured DNS and DHCP  

---

### 2. OU Design & Administration
- Created segmented OU structure:
  - Users  
  - Computers  
  - Groups  
  - Admins  
- Implemented delegated control for Help Desk  

---

### 3. PowerShell Automation
- Generated 1,000+ users via script  
- Standardized usernames and attributes  
- Verified with AD queries  

---

### 4. RBAC Implementation
- Created role-based groups:
  - IT Admins  
  - Help Desk  
  - Standard Users  
- Assigned permissions based on role  

---

### 5. Group Policy Security Baselines
Configured:
- Password policies  
- Account lockout  
- Workstation restrictions  
- Audit logging  

Validated using:
- `gpresult`  
- GPO reports  

---

### 6. Client Integration
- Joined Windows 10 machine to domain  
- Verified authentication and policy application  

---

### 7. Troubleshooting Scenarios
- Domain join failures  
- GPO not applying  
- DNS misconfiguration  
- Account lockouts  

---

### 8. Cloud Identity (Microsoft Entra ID)
- Created users and groups in Microsoft Entra ID  
- Enabled multi-factor authentication (MFA)  
- Explored Conditional Access concepts  

> **Note:** This was implemented as a standalone extension to understand cloud identity models alongside on-prem Active Directory.
