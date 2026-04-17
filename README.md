# Active Directory Systems Administration Lab

A hands-on Windows Server Active Directory lab built on Oracle VirtualBox simulating enterprise identity and infrastructure administration. This lab covers domain controller deployment, OU design, Group Policy security baselines, PowerShell automation, DHCP/DNS configuration, RBAC, and real-world troubleshooting scenarios — mirroring the tasks performed daily in enterprise sysadmin roles.

---

## Lab Environment

| Component | Details |
|---|---|
| Host OS | Windows 11 |
| Virtualization | Oracle VirtualBox |
| Domain Controller OS | Windows Server 2019 |
| Client OS | Windows 10 Pro |
| Domain Name | mydomain.com |
| DC Hostname | DC |
| Client Hostname | CLIENT1 |

### Network Design

| Adapter | Type | Purpose |
|---|---|---|
| Adapter 1 | NAT | Internet access for Domain Controller |
| Adapter 2 | Internal Network (intnet) | Internal domain traffic between DC and clients |

### IP Scheme

| Device | IP Address |
|---|---|
| Domain Controller (_INTERNAL) | 172.16.0.1 |
| DHCP Scope | 172.16.0.100 – 172.16.0.200 |
| DNS | 127.0.0.1 (DC points to itself) |

---

## Table of Contents

1. [Software & ISOs](#1-software--isos)
2. [Domain Controller VM Setup](#2-domain-controller-vm-setup)
3. [Active Directory Installation & Promotion](#3-active-directory-installation--promotion)
4. [OU Structure & Admin Account](#4-ou-structure--admin-account)
5. [NAT & DHCP Configuration](#5-nat--dhcp-configuration)
6. [DNS Verification](#6-dns-verification)
7. [PowerShell User Provisioning](#7-powershell-user-provisioning)
8. [Group Policy Security Baselines](#8-group-policy-security-baselines)
9. [RBAC — Role Based Access Control](#9-rbac--role-based-access-control)
10. [Client VM Setup & Domain Join](#10-client-vm-setup--domain-join)
11. [Troubleshooting Scenarios](#11-troubleshooting-scenarios)

---

## 1. Software & ISOs

### Oracle VirtualBox
Download and install from [virtualbox.org](https://www.virtualbox.org/wiki/Downloads):
- Windows Hosts package
- Extension Pack

### ISOs Required
- [Windows Server 2019](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019)
- [Windows 10](https://www.microsoft.com/en-us/software-download/windows10)

---

## 2. Domain Controller VM Setup

Create a new VM in VirtualBox:

- Name: `Domain Controller`
- RAM: 4096 MB
- CPU: 4 cores
- Disk: 50 GB (dynamically allocated)
- Clipboard & Drag and Drop: Bidirectional
- Network Adapter 1: NAT
- Network Adapter 2: Internal Network (`intnet`)

Mount the Windows Server 2019 ISO and start the VM.

During installation select:
- Edition: **Windows Server 2019 Standard Evaluation (Desktop Experience)**
- Install type: **Custom: Install Windows only**

Set Administrator password: `Password1`

After install, rename the PC to `DC` and restart.

### Configure Network Adapters

Rename adapters for clarity:
- NAT adapter → `INTERNET`
- Internal adapter → `_INTERNAL`

Set a static IP on `_INTERNAL`:

```
IP Address:       172.16.0.1
Subnet Mask:      255.255.255.0
Default Gateway:  (leave blank)
Preferred DNS:    127.0.0.1
```

Restart the DC after applying network settings.

---

## 3. Active Directory Installation & Promotion

### Install AD DS Role

Open Server Manager → Add Roles and Features → select **Active Directory Domain Services**

Complete the wizard and install.

### Promote to Domain Controller

Click the warning flag in Server Manager → **Promote this server to a domain controller**

- Deployment operation: **Add a new forest**
- Root domain name: `mydomain.com`
- DSRM password: `Password1`

Complete the wizard. The server will restart automatically.

Sign back in using: `MYDOMAIN\Administrator`

---

## 4. OU Structure & Admin Account

A well-designed OU structure is foundational to delegated administration and GPO targeting in enterprise environments.

### OU Design

Open **Active Directory Users and Computers** and create the following OU structure:

```
mydomain.com
├── _ADMINS
├── _USERS
├── _COMPUTERS
├── _SERVERS
└── _GROUPS
    ├── Security Groups
    └── Distribution Groups
```

Create each OU:
- Right-click domain → New → Organizational Unit
- Uncheck "Protect container from accidental deletion" for lab purposes

### Create a Domain Admin Account

In `_ADMINS`, create a new user:

- First Name: your first name
- Last Name: your last name
- Username: `a-<firstinitial><lastname>` (e.g., `a-jdoe`)
- Password: `Password1`
- Uncheck: User must change password at next logon
- Check: Password never expires

Right-click the user → Properties → Member Of → Add → type `Domain Admins` → OK

Sign out and sign back in using your new domain admin account for all remaining steps.

---

## 5. NAT & DHCP Configuration

### Configure NAT (Routing and Remote Access)

Add the **Remote Access** role with the **Routing** role service included.

Open **Routing and Remote Access** → right-click DC → Configure and Enable:
- Configuration: **NAT**
- Public interface: `INTERNET`

This allows client VMs on the internal network to reach the internet through the DC.

### Configure DHCP

Add the **DHCP Server** role.

Open **DHCP** → IPv4 → New Scope:

| Setting | Value |
|---|---|
| Scope Name | 172.16.0.100-200 |
| Start IP | 172.16.0.100 |
| End IP | 172.16.0.200 |
| Subnet Mask | 255.255.255.0 |
| Router (Gateway) | 172.16.0.1 |
| DNS Server | 172.16.0.1 |
| Domain Name | mydomain.com |

Activate the scope, authorize the DHCP server, and refresh.

---

## 6. DNS Verification

DNS is the most common cause of AD failures. Verifying DNS health before adding clients prevents hard-to-diagnose issues later.

### Verify DNS Zones

Open **DNS Manager** → expand DC → Forward Lookup Zones → confirm `mydomain.com` exists.

Check that the following records exist under `mydomain.com`:
- A record for `DC` pointing to `172.16.0.1`
- SOA and NS records

### Test DNS Resolution from DC

Open PowerShell on the DC:

```powershell
# Resolve domain name
Resolve-DnsName mydomain.com

# Check SRV records (critical for AD client discovery)
Resolve-DnsName _ldap._tcp.mydomain.com -Type SRV

# Run full DC diagnostic
dcdiag /test:dns
```

### Verify AD Replication Health

```powershell
# Check replication status
repadmin /showrepl

# Force replication sync
repadmin /syncall /AdeP

# Run full DC diagnostic
dcdiag /v
```

---

## 7. PowerShell User Provisioning

Automating user account creation with PowerShell mirrors how enterprise environments handle bulk onboarding.

Save the following as `CREATE_USERS.ps1` on the Desktop:

```powershell
$PASSWORD_FOR_USERS = "Password1"
$password = ConvertTo-SecureString $PASSWORD_FOR_USERS -AsPlainText -Force

$firstNames = @("James", "Mary", "John", "Patricia", "Robert", "Jennifer",
                "Michael", "Linda", "William", "Elizabeth", "David", "Barbara",
                "Richard", "Susan", "Joseph", "Jessica", "Thomas", "Sarah",
                "Charles", "Karen")

$lastNames = @("Smith", "Johnson", "Williams", "Brown", "Jones", "Garcia",
               "Miller", "Davis", "Rodriguez", "Martinez", "Hernandez", "Lopez",
               "Gonzalez", "Wilson", "Anderson", "Thomas", "Taylor", "Moore",
               "Jackson", "Martin")

$USER_FIRST_LAST_LIST = for ($i = 0; $i -lt 1000; $i++) {
    "$($firstNames | Get-Random) $($lastNames | Get-Random)"
}

New-ADOrganizationalUnit -Name _USERS -ProtectedFromAccidentalDeletion $false -ErrorAction SilentlyContinue

foreach ($n in $USER_FIRST_LAST_LIST) {
    $first = $n.Split(" ")[0].ToLower()
    $last  = $n.Split(" ")[1].ToLower()
    $username = "$($first.Substring(0,1))$($last)".ToLower()

    Write-Host "Creating user: $username" -ForegroundColor Cyan

    New-ADUser `
        -AccountPassword $password `
        -GivenName $first `
        -Surname $last `
        -DisplayName $username `
        -Name $username `
        -EmployeeID $username `
        -PasswordNeverExpires $true `
        -Path "OU=_USERS,DC=mydomain,DC=com" `
        -Enabled $true
}

Write-Host "User provisioning complete." -ForegroundColor Green
```

Run in PowerShell ISE as Administrator:

```powershell
Set-ExecutionPolicy Unrestricted
# Open and run CREATE_USERS.ps1
```

### Verify Users Were Created

```powershell
# Count users in _USERS OU
(Get-ADUser -Filter * -SearchBase "OU=_USERS,DC=mydomain,DC=com").Count

# View a sample user
Get-ADUser -Identity jsmith -Properties *
```

---

## 8. Group Policy Security Baselines

Group Policy is one of the most critical tools in enterprise Windows administration. This section documents security baseline GPOs that mirror real-world hardening standards.

### Password Policy GPO

Open **Group Policy Management** → right-click `mydomain.com` → Create a GPO → name it `Security Baseline - Password Policy`

Edit the GPO:

```
Computer Configuration →
  Policies →
    Windows Settings →
      Security Settings →
        Account Policies →
          Password Policy:
            Enforce password history:          10 passwords
            Maximum password age:              90 days
            Minimum password age:              1 day
            Minimum password length:           12 characters
            Password must meet complexity:     Enabled
            Store passwords using reversible encryption: Disabled

          Account Lockout Policy:
            Account lockout threshold:         5 invalid attempts
            Account lockout duration:          30 minutes
            Reset account lockout counter:     15 minutes
```

### Workstation Hardening GPO

Create a new GPO named `Security Baseline - Workstation Hardening` and link it to `_COMPUTERS` OU:

```
Computer Configuration →
  Policies →
    Windows Settings →
      Security Settings →
        Local Policies →
          Security Options:
            Interactive logon: Do not display last user name: Enabled
            Accounts: Rename administrator account: localadmin
            Shutdown: Allow system to be shut down without logging on: Disabled

        Event Log:
          Maximum application log size: 32768 KB
          Maximum security log size:    81920 KB
          Maximum system log size:      32768 KB

  Administrative Templates →
    System →
      Removable Storage Access:
        All Removable Storage classes - Deny all access: Enabled
    Windows Components →
      Windows Update:
        Configure Automatic Updates: Enabled (Auto download and schedule install)
```

### Audit Policy GPO

Create a GPO named `Security Baseline - Audit Policy`:

```
Computer Configuration →
  Policies →
    Windows Settings →
      Security Settings →
        Advanced Audit Policy Configuration →
          Account Logon:
            Audit Credential Validation: Success and Failure
          Account Management:
            Audit User Account Management: Success and Failure
          Logon/Logoff:
            Audit Logon: Success and Failure
            Audit Logoff: Success
          Object Access:
            Audit File System: Failure
          Policy Change:
            Audit Policy Change: Success and Failure
          Privilege Use:
            Audit Sensitive Privilege Use: Success and Failure
          System:
            Audit Security System Extension: Success and Failure
```

### Force GPO Update and Verify

```powershell
# Force immediate GPO refresh on DC
gpupdate /force

# Verify applied GPOs
gpresult /r

# Generate detailed HTML GPO report
gpresult /h C:\GPOReport.html
```

---

## 9. RBAC — Role Based Access Control

Role-based access control ensures users have only the permissions required for their job function — a core principle of least-privilege security.

### Create Security Groups

```powershell
# Create role-based security groups in _GROUPS OU
New-ADGroup -Name "IT-Admins" -GroupScope Global -GroupCategory Security `
    -Path "OU=_GROUPS,DC=mydomain,DC=com" -Description "Full IT administrative access"

New-ADGroup -Name "Help-Desk" -GroupScope Global -GroupCategory Security `
    -Path "OU=_GROUPS,DC=mydomain,DC=com" -Description "Tier 1 and 2 support access"

New-ADGroup -Name "Standard-Users" -GroupScope Global -GroupCategory Security `
    -Path "OU=_GROUPS,DC=mydomain,DC=com" -Description "Standard employee access"
```

### Assign Users to Groups

```powershell
# Add a user to IT-Admins
Add-ADGroupMember -Identity "IT-Admins" -Members "a-jdoe"

# Add bulk users to Standard-Users
Get-ADUser -Filter * -SearchBase "OU=_USERS,DC=mydomain,DC=com" |
    ForEach-Object { Add-ADGroupMember -Identity "Standard-Users" -Members $_ }

# Verify group membership
Get-ADGroupMember -Identity "IT-Admins"
```

### Delegate OU Administration

Delegating control allows the Help Desk group to manage user accounts in `_USERS` without full Domain Admin rights:

- Right-click `_USERS` OU → Delegate Control
- Add group: `Help-Desk`
- Delegate: Reset user passwords and force password change at next logon

---

## 10. Client VM Setup & Domain Join

### Create CLIENT1 VM

- Name: `CLIENT1`
- RAM: 4096 MB
- CPU: 4 cores
- Disk: 50 GB
- Network: Internal Network (`intnet`) only
- Mount Windows 10 ISO

During installation:
- Edition: **Windows 10 Pro**
- Select: I don't have internet → Continue with limited setup
- Username: `user`, no password
- Skip all optional settings

### Verify Network Connectivity

Open Command Prompt on CLIENT1:

```cmd
ipconfig /all
ping 172.16.0.1
ping www.google.com
```

If Default Gateway or DNS is missing:
- On DC: verify DHCP scope options (Router and DNS server entries)
- On CLIENT1: run `ipconfig /renew`

### Join Domain

Right-click Start → System → Rename this PC (Advanced):
- Computer name: `CLIENT1`
- Member of Domain: `mydomain.com`
- Enter domain admin credentials: `a-jdoe` / `Password1`
- Restart

After restart, sign in via **Other user** using a domain account from `_USERS`.

### Verify in Active Directory

On the DC:
- **DHCP** → IPv4 → Address Leases → confirm `CLIENT1` listed
- **Active Directory Users and Computers** → `_COMPUTERS` → confirm `CLIENT1` listed

Move CLIENT1 to the `_COMPUTERS` OU:

```powershell
Get-ADComputer -Identity CLIENT1 | Move-ADObject -TargetPath "OU=_COMPUTERS,DC=mydomain,DC=com"
```

---

## 11. Troubleshooting Scenarios

This section documents real failure scenarios with diagnosis steps and resolutions — simulating Level 2 and Level 3 incident response in an AD environment.

---

### Scenario 1 — Client Cannot Join Domain

**Symptom:** Domain join fails with "domain not found" or "domain controller not available."

**Diagnosis:**

```cmd
# On CLIENT1
ping 172.16.0.1
nslookup mydomain.com
ipconfig /all
```

**Common Root Causes and Fixes:**

| Root Cause | Fix |
|---|---|
| Wrong DNS server on client | Set DNS to 172.16.0.1 manually |
| DHCP scope DNS option not set | Add DNS server 172.16.0.1 in DHCP scope options |
| DC firewall blocking traffic | Verify Windows Firewall allows domain traffic on DC |
| Wrong network adapter on CLIENT1 | Confirm CLIENT1 uses Internal Network, not NAT |

---

### Scenario 2 — GPO Not Applying to Client

**Symptom:** A GPO is linked and enabled but settings are not applying on CLIENT1.

**Diagnosis:**

```cmd
# On CLIENT1 — force GPO refresh and check results
gpupdate /force
gpresult /r
gpresult /h C:\GPOReport.html
```

```powershell
# On DC — check GPO link and enforcement
Get-GPO -Name "Security Baseline - Password Policy"
Get-GPInheritance -Target "OU=_COMPUTERS,DC=mydomain,DC=com"
```

**Common Root Causes and Fixes:**

| Root Cause | Fix |
|---|---|
| GPO linked to wrong OU | Confirm CLIENT1 is in `_COMPUTERS` OU, not default Computers container |
| GPO not enforced | Enable Enforced on the GPO link |
| Security filtering excluding machine | Verify Authenticated Users has Read and Apply Group Policy |
| Replication not complete | Run `repadmin /syncall /AdeP` on DC |

---

### Scenario 3 — AD Replication Failure

**Symptom:** Changes made on the DC are not reflecting, or `dcdiag` reports replication errors.

**Diagnosis:**

```powershell
# Check replication status
repadmin /showrepl

# Show replication summary
repadmin /replsummary

# Run full DC health check
dcdiag /v

# Check event logs for replication errors
Get-EventLog -LogName "Directory Service" -EntryType Error -Newest 20
```

**Resolution:**

```powershell
# Force replication sync
repadmin /syncall /AdeP

# If KCC (Knowledge Consistency Checker) is the issue, force recalculation
repadmin /kcc
```

---

### Scenario 4 — User Account Locked Out

**Symptom:** User reports they cannot log in. Help Desk confirms the account is locked.

**Diagnosis:**

```powershell
# Check account status
Get-ADUser -Identity jsmith -Properties LockedOut, BadLogonCount, LastBadPasswordAttempt |
    Select-Object Name, LockedOut, BadLogonCount, LastBadPasswordAttempt

# Find which DC locked the account
Get-WinEvent -ComputerName DC -FilterHashtable @{
    LogName = 'Security'
    Id = 4740
} -MaxEvents 10 | Select-Object TimeCreated, Message
```

**Resolution:**

```powershell
# Unlock account
Unlock-ADAccount -Identity jsmith

# Verify
Get-ADUser -Identity jsmith -Properties LockedOut | Select-Object Name, LockedOut

# Force password reset at next logon if needed
Set-ADUser -Identity jsmith -ChangePasswordAtLogon $true
```

---

### Scenario 5 — DNS Resolution Failure

**Symptom:** Clients cannot resolve domain resources. `ping mydomain.com` fails.

**Diagnosis:**

```powershell
# On DC
dcdiag /test:dns /v

# Check DNS service status
Get-Service -Name DNS

# Review DNS event log
Get-EventLog -LogName "DNS Server" -EntryType Error -Newest 10
```

**Resolution:**

```powershell
# Restart DNS service if stopped
Restart-Service -Name DNS

# Clear DNS cache
Clear-DnsServerCache

# Register DC's own DNS record
ipconfig /registerdns

# Verify forward lookup zone is intact
Get-DnsServerZone -Name "mydomain.com"
```

---

## Tools Reference

| Tool | Purpose |
|---|---|
| dcdiag | Domain controller health and diagnostic testing |
| repadmin | AD replication status and synchronization |
| gpupdate | Force Group Policy refresh |
| gpresult | View applied GPOs and resultant set of policy |
| Get-ADUser | Query and manage AD user accounts via PowerShell |
| Get-ADGroupMember | View group membership |
| Get-WinEvent | Query Windows Event Logs via PowerShell |
| nslookup | DNS resolution testing |
| ipconfig /renew | Refresh DHCP lease on client |
| Move-ADObject | Move AD objects between OUs |

---

[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:kennymiranda000@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/kenneth-miranda-xyz)
