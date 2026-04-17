# Enterprise Active Directory & Identity Lab

A hands-on Active Directory lab built on Windows Server 2022 simulating 
enterprise identity and systems administration. This lab covers domain 
deployment, OU design, delegated administration, RBAC, PowerShell automation 
at scale, Group Policy security baselines, and real-world troubleshooting 
using Event Viewer and PowerShell diagnostics. The environment was extended 
with Microsoft Entra ID to reflect modern hybrid identity practices.

---

## Lab Environment

| Component         | Details                              |
|-------------------|--------------------------------------|
| Host OS           | Windows 11 (dual boot)               |
| Virtualization    | Oracle VirtualBox                    |
| Domain Controller | Windows Server 2022 (DC)             |
| Client Machine    | Windows 10 Pro                       |
| Domain Name       | lab.local                            |
| Network           | Internal VirtualBox network + DHCP   |
| Cloud Identity    | Microsoft Entra ID (free tenant)     |

---

## Table of Contents

1. [Domain Controller Deployment](#1-domain-controller-deployment)
2. [OU Design & Delegated Administration](#2-ou-design--delegated-administration)
3. [PowerShell Automation](#3-powershell-automation)
4. [RBAC Implementation](#4-rbac-implementation)
5. [Group Policy Security Baselines](#5-group-policy-security-baselines)
6. [Client Integration](#6-client-integration)
7. [Microsoft Entra ID](#7-microsoft-entra-id)
8. [Troubleshooting Scenarios](#8-troubleshooting-scenarios)

---

## 1. Domain Controller Deployment

### Install Windows Server 2022

Create a new VM in VirtualBox:

- Name: `DC01`
- Type: Windows / Windows 2022 (64-bit)
- RAM: 2048 MB | CPU: 2 cores | Disk: 50 GB (dynamically allocated)

Assign a static IP before promoting to domain controller:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.56.10 `
  -PrefixLength 24 -DefaultGateway 192.168.56.1

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 127.0.0.1
```

### Install AD DS Role

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

### Promote to Domain Controller

```powershell
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetBIOSName "LAB" `
  -InstallDns:$true `
  -Force:$true
```

The server will restart automatically after promotion.

### Verify Domain and DNS

```powershell
Get-ADDomain
Get-ADDomainController
Resolve-DnsName lab.local
```

---

## 2. OU Design & Delegated Administration

### OU Structure

The domain was segmented into OUs reflecting a real enterprise layout, 
separating administrative accounts from standard users, and organizing 
computers and groups independently for targeted GPO application:

```
lab.local
├── Corp
│   ├── Users
│   │   ├── IT
│   │   ├── HR
│   │   ├── Finance
│   │   └── Sales
│   ├── Computers
│   │   ├── Workstations
│   │   └── Servers
│   ├── Groups
│   └── Admins
│       └── HelpDesk
```

### Create OU Structure via PowerShell

```powershell
$base = "DC=lab,DC=local"

New-ADOrganizationalUnit -Name "Corp" -Path $base
New-ADOrganizationalUnit -Name "Users" -Path "OU=Corp,$base"
New-ADOrganizationalUnit -Name "IT"    -Path "OU=Users,OU=Corp,$base"
New-ADOrganizationalUnit -Name "HR"    -Path "OU=Users,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Finance" -Path "OU=Users,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Sales" -Path "OU=Users,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Computers" -Path "OU=Corp,$base"
New-ADOrganizationalUnit -Name "Workstations" -Path "OU=Computers,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Servers" -Path "OU=Computers,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Groups" -Path "OU=Corp,$base"
New-ADOrganizationalUnit -Name "Admins" -Path "OU=Corp,$base"
New-ADOrganizationalUnit -Name "HelpDesk" -Path "OU=Admins,OU=Corp,$base"
```

### Delegated Administration — Help Desk

Help Desk staff were granted delegated control over the Users OU, limited 
to password resets and account unlocks without granting full Domain Admin 
rights:

```powershell
# Delegate password reset to HelpDesk group on the Users OU
$helpdesk = Get-ADGroup "HelpDesk"
$usersOU  = "OU=Users,OU=Corp,DC=lab,DC=local"

# Using the Delegation of Control Wizard (GUI) for granular ACL assignment,
# then verifying the resulting ACL via PowerShell:
(Get-Acl "AD:$usersOU").Access | Where-Object {
  $_.IdentityReference -like "*HelpDesk*"
} | Select-Object IdentityReference, ActiveDirectoryRights
```

Permissions delegated to HelpDesk:

- Reset passwords on user objects
- Read/write `lockoutTime` (unlock accounts)
- Read all user attributes

---

## 3. PowerShell Automation

### Bulk User Provisioning with Role-Based Group Mapping

1,000+ users were generated from a CSV file. Each user's `Department` 
field drove automatic assignment to the corresponding role-based security 
group and OU, eliminating manual account creation and enforcing consistent 
group membership at provisioning time.

**users.csv sample:**

```
FirstName,LastName,Department,Title
James,Carter,IT,Systems Administrator
Maria,Lopez,HR,HR Coordinator
David,Kim,Finance,Financial Analyst
Sarah,Nguyen,Sales,Account Executive
```

**Provisioning script:**

```powershell
# Department-to-OU and group mapping table
$roleMap = @{
  "IT"      = @{ OU = "OU=IT,OU=Users,OU=Corp,DC=lab,DC=local";
                  Group = "IT-Staff" }
  "HR"      = @{ OU = "OU=HR,OU=Users,OU=Corp,DC=lab,DC=local";
                  Group = "HR-Staff" }
  "Finance" = @{ OU = "OU=Finance,OU=Users,OU=Corp,DC=lab,DC=local";
                  Group = "Finance-Staff" }
  "Sales"   = @{ OU = "OU=Sales,OU=Users,OU=Corp,DC=lab,DC=local";
                  Group = "Sales-Staff" }
}

$users = Import-Csv -Path "C:\Scripts\users.csv"

foreach ($user in $users) {
  $username   = ($user.FirstName[0] + $user.LastName).ToLower()
  $fullName   = "$($user.FirstName) $($user.LastName)"
  $department = $user.Department
  $targetOU   = $roleMap[$department].OU
  $targetGroup = $roleMap[$department].Group

  # Skip if department not in map
  if (-not $roleMap.ContainsKey($department)) {
    Write-Warning "Unknown department '$department' for $fullName — skipping"
    continue
  }

  # Create user account
  New-ADUser `
    -Name            $fullName `
    -GivenName       $user.FirstName `
    -Surname         $user.LastName `
    -SamAccountName  $username `
    -UserPrincipalName "$username@lab.local" `
    -Title           $user.Title `
    -Department      $department `
    -Path            $targetOU `
    -AccountPassword (ConvertTo-SecureString "Welcome@1234!" -AsPlainText -Force) `
    -ChangePasswordAtLogon $true `
    -Enabled         $true

  # Assign to role-based group based on department
  Add-ADGroupMember -Identity $targetGroup -Members $username

  Write-Host "Created: $username | OU: $department | Group: $targetGroup"
}
```

### Verify Provisioning

```powershell
# Count users per OU
Get-ADUser -Filter * -SearchBase "OU=IT,OU=Users,OU=Corp,DC=lab,DC=local" |
  Measure-Object

# Confirm group membership for a sample user
Get-ADUser "jcarter" -Properties MemberOf |
  Select-Object -ExpandProperty MemberOf

# Spot-check department attribute
Get-ADUser "jcarter" -Properties Department, Title |
  Select-Object Name, Department, Title
```

---

## 4. RBAC Implementation

### Create Role-Based Security Groups

```powershell
$groupOU = "OU=Groups,OU=Corp,DC=lab,DC=local"

New-ADGroup -Name "IT-Staff"      -GroupScope Global -Path $groupOU
New-ADGroup -Name "HR-Staff"      -GroupScope Global -Path $groupOU
New-ADGroup -Name "Finance-Staff" -GroupScope Global -Path $groupOU
New-ADGroup -Name "Sales-Staff"   -GroupScope Global -Path $groupOU
New-ADGroup -Name "HelpDesk"      -GroupScope Global -Path $groupOU
New-ADGroup -Name "IT-Admins"     -GroupScope Global -Path $groupOU
```

### Role Permissions Summary

| Group         | Rights                                               |
|---------------|------------------------------------------------------|
| IT-Admins     | Domain Admins (full control)                         |
| IT-Staff      | Remote Desktop access, software installation rights  |
| HelpDesk      | Delegated: password reset, account unlock on Users OU|
| HR-Staff      | Access to HR file share only                         |
| Finance-Staff | Access to Finance file share only                    |
| Sales-Staff   | Access to Sales file share only                      |

### Verify Group Membership

```powershell
Get-ADGroupMember -Identity "IT-Staff" | Select-Object Name, SamAccountName
Get-ADGroupMember -Identity "HelpDesk" | Select-Object Name, SamAccountName
```

---

## 5. Group Policy Security Baselines

All GPOs were linked at the `OU=Corp` level unless otherwise noted, 
and validated using `gpresult` and Event Viewer.

### Password & Account Lockout Policy

Applied via Default Domain Policy:

| Setting                        | Value     |
|--------------------------------|-----------|
| Minimum password length        | 12 chars  |
| Password complexity            | Enabled   |
| Maximum password age           | 90 days   |
| Account lockout threshold      | 5 attempts|
| Lockout duration               | 30 minutes|
| Reset lockout counter after    | 15 minutes|

```
GPO Path:
Computer Configuration > Policies > Windows Settings >
  Security Settings > Account Policies
```

### Workstation Hardening GPO

```
GPO Name: Workstation-Hardening
Linked to: OU=Workstations,OU=Computers,OU=Corp

Settings applied:
Computer Configuration > Policies > Windows Settings > Security Settings:
  - Disable Guest account
  - Rename Administrator account to a non-default name
  - Interactive logon: Do not display last signed-in user name — Enabled
  - Limit local administrators to domain IT-Admins group only

Computer Configuration > Administrative Templates > System:
  - Prevent access to registry editing tools — Enabled
  - Prevent access to the command prompt — Enabled (Standard Users)

Computer Configuration > Administrative Templates > Windows Components >
  Windows Defender:
  - Turn off Windows Defender — Disabled (keep Defender active)
```

### Software Restriction Policy

Restricts execution of unauthorized software on standard user workstations:

```
GPO Name: Software-Restrictions
Linked to: OU=Workstations,OU=Computers,OU=Corp

Computer Configuration > Policies > Windows Settings > Security Settings >
  Software Restriction Policies:

  - Default Security Level: Disallowed
  - Additional Rules (Unrestricted):
      %WINDIR%\**         — allow Windows system binaries
      %PROGRAMFILES%\**   — allow installed applications
      %PROGRAMFILES(x86)%\** — allow 32-bit installed applications

  Effect: Executables run from Downloads, Desktop, %TEMP%, or 
  removable media are blocked for standard users.
```

Validate that the policy is applying:

```powershell
gpresult /r /scope computer
gpresult /h C:\Reports\gpo-report.html; Start-Process gpo-report.html
```

### Audit Logging GPO

```
GPO Name: Audit-Logging-Baseline
Linked to: OU=Corp

Computer Configuration > Policies > Windows Settings > Security Settings >
  Advanced Audit Policy Configuration:

  - Audit Account Logon Events    — Success, Failure
  - Audit Account Management      — Success, Failure
  - Audit Logon Events            — Success, Failure
  - Audit Object Access           — Failure
  - Audit Policy Change           — Success
  - Audit Privilege Use           — Failure
  - Audit System Events           — Success, Failure
```

Verify audit events are being written:

```powershell
# View recent logon failures (Event ID 4625)
Get-WinEvent -FilterHashtable @{
  LogName   = 'Security'
  Id        = 4625
  StartTime = (Get-Date).AddHours(-1)
} | Select-Object TimeCreated, Message | Format-List
```

---

## 6. Client Integration

### Join Windows 10 to Domain

On the Windows 10 VM, set DNS to point to the DC:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.56.10
```

Join the domain:

```powershell
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

Move the computer object to the correct OU after joining:

```powershell
# Run on DC
Get-ADComputer "WIN10-CLIENT" | Move-ADObject `
  -TargetPath "OU=Workstations,OU=Computers,OU=Corp,DC=lab,DC=local"
```

### Verify Policy Application on Client

```powershell
# Force policy refresh
gpupdate /force

# Confirm applied GPOs
gpresult /r

# View applied GPOs and their details
gpresult /h C:\Reports\client-gpo-report.html
Start-Process C:\Reports\client-gpo-report.html
```

---

## 7. Microsoft Entra ID

Microsoft Entra ID was configured as a standalone cloud identity extension 
to explore hybrid identity concepts alongside the on-premises AD environment.

### Create Users and Groups

Users and groups were created in the Entra ID portal and via PowerShell 
using the Microsoft Graph module:

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All"

# Create a cloud user
New-MgUser -DisplayName "Test User" `
  -UserPrincipalName "testuser@.onmicrosoft.com" `
  -MailNickname "testuser" `
  -AccountEnabled `
  -PasswordProfile @{
    Password                      = "TempPass@2024!"
    ForceChangePasswordNextSignIn = $true
  }
```

### Enable Multi-Factor Authentication

MFA was enforced per-user via the Entra ID portal:

```
Azure Portal > Microsoft Entra ID > Users >
  Per-user MFA > Select user > Enable MFA
```

Verified by signing in as the test user and confirming MFA prompt 
(Microsoft Authenticator app).

### Conditional Access Policy

A basic Conditional Access policy was configured to require MFA for 
all users signing in from outside a trusted location:

```
Azure Portal > Microsoft Entra ID > Security > Conditional Access >
  New Policy:

  Name: Require MFA — All Users
  Assignments:
    Users: All users (excluding break-glass admin account)
    Cloud apps: All cloud apps
    Conditions:
      Locations: Any location
        Exclude: Trusted location (home lab public IP)
  Access controls:
    Grant: Require multi-factor authentication

  Policy state: Report-only (for testing before enforcement)
```

> **Note:** This was implemented as a standalone extension using a free 
> Microsoft Entra ID tenant to understand cloud identity models alongside 
> on-premises Active Directory instead of a live sync with the lab domain.

---

## 8. Troubleshooting Scenarios

This section documents real failure scenarios reproduced in the lab, 
how they were diagnosed, and how they were resolved. All diagnosis 
involved a combination of Event Viewer and PowerShell, matching the 
workflow used in real enterprise helpdesk and junior sysadmin roles.

---

### Scenario 1 — GPO Not Applying to Client

**Symptom:** A workstation hardening policy is not taking effect on 
the Windows 10 client after being linked to the Workstations OU.

**Diagnosis:**

```powershell
# Run on the affected client
gpresult /r

# Check for GPO errors in Event Viewer
Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" |
  Where-Object { $_.LevelDisplayName -eq "Error" -or
                 $_.LevelDisplayName -eq "Warning" } |
  Select-Object TimeCreated, Id, Message | Format-List
```

Open Event Viewer manually and navigate to:

```
Event Viewer > Applications and Services Logs >
  Microsoft > Windows > GroupPolicy > Operational
```

Look for Event ID `7016` (policy processing error) or `1085` 
(policy failed to apply).

**Root Cause:** The computer object was still in the default `Computers` 
container, not in the `OU=Workstations` OU where the GPO was linked.

**Resolution:**

```powershell
# Move computer to correct OU
Get-ADComputer "WIN10-CLIENT" | Move-ADObject `
  -TargetPath "OU=Workstations,OU=Computers,OU=Corp,DC=lab,DC=local"

# Force policy refresh on client
gpupdate /force

# Verify GPO now appears in applied list
gpresult /r
```

---

### Scenario 2 — DNS Misconfiguration Breaks Domain Join

**Symptom:** Attempting to join a workstation to `lab.local` fails with 
"The specified domain either does not exist or could not be contacted."

**Diagnosis:**

```powershell
# Test DNS resolution from the client
Resolve-DnsName lab.local
nslookup lab.local

# Verify DNS server address on the client NIC
Get-DnsClientServerAddress

# Test DC connectivity
Test-NetConnection -ComputerName 192.168.56.10 -Port 389
```

Check DNS server logs on the DC via Event Viewer:

```
Event Viewer > Windows Logs > DNS Server
```

Look for Event ID `4013` (DNS server waiting for AD) or `4015` 
(DNS server error).

**Root Cause:** The Windows 10 client's DNS was pointing to the 
VirtualBox NAT gateway instead of the DC at `192.168.56.10`.

**Resolution:**

```powershell
# Fix DNS on the client
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.56.10

# Flush DNS cache
ipconfig /flushdns

# Retry domain join
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

---

### Scenario 3 — Account Lockout

**Symptom:** A user cannot log in. Reports their password is correct.

**Diagnosis:**

```powershell
# Check lockout status
Get-ADUser "jcarter" -Properties LockedOut, BadLogonCount,
  BadPasswordTime, LastLogonDate |
  Select-Object Name, LockedOut, BadLogonCount, BadPasswordTime

# Find the source of the lockouts using Security event log
Get-WinEvent -FilterHashtable @{
  LogName   = 'Security'
  Id        = 4740
} | Select-Object TimeCreated, Message | Format-List
```

In Event Viewer, Event ID `4740` (account locked out) will show the 
caller computer name, identifying which machine is triggering the 
failed authentications (e.g., a saved stale credential on a shared 
workstation).

**Root Cause:** A stale cached credential on a shared workstation was 
repeatedly authenticating with an old password after a password change.

**Resolution:**

```powershell
# Unlock the account
Unlock-ADAccount -Identity "jcarter"

# Confirm
Get-ADUser "jcarter" -Properties LockedOut |
  Select-Object Name, LockedOut
```

Clear the stale credential from the source machine:

```
Control Panel > Credential Manager > Windows Credentials >
  Remove the cached lab.local entry
```

---

### Scenario 4 — Access Control Issue (Broken File Share Permissions)

**Symptom:** A user in the `Finance-Staff` group cannot access the 
Finance file share despite being a group member.

**Diagnosis:**

```powershell
# Verify group membership
Get-ADUser "mlopez" -Properties MemberOf |
  Select-Object -ExpandProperty MemberOf

# Check effective NTFS permissions on the share folder
Get-Acl "C:\Shares\Finance" | Format-List

# Check share permissions specifically
Get-SmbShareAccess -Name "Finance"
```

Check for access denied events in Event Viewer:

```
Event Viewer > Windows Logs > Security
Filter for Event ID 4663 (object access attempt) and
Event ID 5145 (network share access check)
```

**Root Cause:** The NTFS permissions on `C:\Shares\Finance` granted 
access to `Finance-Staff`, but the SMB share permission was locked 
down to `IT-Admins` only, blocking all other users at the share 
layer before NTFS permissions were even evaluated.

**Resolution:**

```powershell
# Grant Finance-Staff read/change access at the share level
Grant-SmbShareAccess -Name "Finance" -AccountName "LAB\Finance-Staff" `
  -AccessRight Change -Force

# Verify share and NTFS permissions are now aligned
Get-SmbShareAccess -Name "Finance"
Get-Acl "C:\Shares\Finance" | Select-Object -ExpandProperty Access
```

---

### Scenario 5 — Domain Join Failure Due to Duplicate Computer Object

**Symptom:** Rejoining a reimaged workstation to the domain fails with 
"The account already exists."

**Diagnosis:**

```powershell
# Check for existing computer object
Get-ADComputer "WIN10-CLIENT"

# Check last logon to determine if it's a stale object
Get-ADComputer "WIN10-CLIENT" -Properties LastLogonDate |
  Select-Object Name, LastLogonDate
```

Event Viewer on the DC:

```
Event Viewer > Windows Logs > System
Event ID 5722 — The session setup from the computer WIN10-CLIENT failed.
```

**Root Cause:** A stale computer account from the previous installation 
remained in AD. The new machine could not create a new object with the 
same name.

**Resolution:**

```powershell
# Remove the stale computer object
Remove-ADComputer -Identity "WIN10-CLIENT" -Confirm:$false

# Rejoin from the workstation
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

---

## Tools Used

| Tool                    | Purpose                                               |
|-------------------------|-------------------------------------------------------|
| Active Directory Users  | GUI-based user, group, and OU management              |
| and Computers (ADUC)    |                                                       |
| PowerShell AD module    | Bulk provisioning, querying, and automation           |
| Group Policy Management | GPO creation, linking, and reporting                  |
| Console (GPMC)          |                                                       |
| gpresult                | Verify applied GPOs on client and computer            |
| gpupdate                | Force policy refresh                                  |
| Event Viewer            | Diagnose GPO failures, logon events, lockouts,        |
|                         | DNS errors, and access denials                        |
| Get-WinEvent            | PowerShell-based event log querying                   |
| Get-ADUser / Get-ADGroup| AD object inspection and membership verification      |
| Test-NetConnection      | Port and connectivity testing                         |
| Resolve-DnsName         | DNS resolution testing                                |
| Get-Acl / Get-SmbShare  | NTFS and share permission inspection                  |
| Microsoft Entra ID      | Cloud identity, MFA, and Conditional Access           |
| Microsoft Graph PS      | Cloud user and group management via PowerShell        |

---

[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:kennymiranda000@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/kenneth-miranda-xyz)
