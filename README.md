# Enterprise Active Directory & Identity Lab

A hands-on Active Directory lab built on Windows Server 2022 simulating enterprise identity and systems administration. This lab covers domain deployment, OU design, delegated administration, RBAC, PowerShell automation at scale, Group Policy security baselines, patch management with WSUS, web server administration with IIS, certificate services with ADCS, cloud monitoring with Azure Monitor and Log Analytics, and real-world troubleshooting using Event Viewer and PowerShell diagnostics. The environment is extended with Microsoft Entra ID to reflect modern hybrid identity practices.

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
7. [Patch Management with WSUS](#7-patch-management-with-wsus)
8. [Web Server Administration with IIS](#8-web-server-administration-with-iis)
9. [Certificate Services with ADCS](#9-certificate-services-with-adcs)
10. [Microsoft Entra ID](#10-microsoft-entra-id)
11. [Cloud Monitoring with Azure Monitor & Log Analytics](#11-cloud-monitoring-with-azure-monitor--log-analytics)
12. [Troubleshooting Scenarios](#12-troubleshooting-scenarios)

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

The domain was segmented into OUs reflecting a real enterprise layout, separating administrative accounts from standard users, and organizing computers and groups independently for targeted GPO application:

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

New-ADOrganizationalUnit -Name "Corp"      -Path $base
New-ADOrganizationalUnit -Name "Users"     -Path "OU=Corp,$base"
New-ADOrganizationalUnit -Name "IT"        -Path "OU=Users,OU=Corp,$base"
New-ADOrganizationalUnit -Name "HR"        -Path "OU=Users,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Finance"   -Path "OU=Users,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Sales"     -Path "OU=Users,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Computers" -Path "OU=Corp,$base"
New-ADOrganizationalUnit -Name "Workstations" -Path "OU=Computers,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Servers"   -Path "OU=Computers,OU=Corp,$base"
New-ADOrganizationalUnit -Name "Groups"    -Path "OU=Corp,$base"
New-ADOrganizationalUnit -Name "Admins"    -Path "OU=Corp,$base"
New-ADOrganizationalUnit -Name "HelpDesk"  -Path "OU=Admins,OU=Corp,$base"
```

### Delegated Administration — Help Desk

Help Desk staff were granted delegated control over the Users OU, limited to password resets and account unlocks without granting full Domain Admin rights:

```powershell
$helpdesk = Get-ADGroup "HelpDesk"
$usersOU  = "OU=Users,OU=Corp,DC=lab,DC=local"

# Verify delegated ACL after applying via Delegation of Control Wizard
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

1,000+ users were generated from a CSV file. Each user's `Department` field drove automatic assignment to the corresponding role-based security group and OU, eliminating manual account creation and enforcing consistent group membership at provisioning time.

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
$roleMap = @{
  "IT"      = @{ OU = "OU=IT,OU=Users,OU=Corp,DC=lab,DC=local";      Group = "IT-Staff" }
  "HR"      = @{ OU = "OU=HR,OU=Users,OU=Corp,DC=lab,DC=local";      Group = "HR-Staff" }
  "Finance" = @{ OU = "OU=Finance,OU=Users,OU=Corp,DC=lab,DC=local"; Group = "Finance-Staff" }
  "Sales"   = @{ OU = "OU=Sales,OU=Users,OU=Corp,DC=lab,DC=local";   Group = "Sales-Staff" }
}

$users = Import-Csv -Path "C:\Scripts\users.csv"

foreach ($user in $users) {
  $username    = ($user.FirstName[0] + $user.LastName).ToLower()
  $fullName    = "$($user.FirstName) $($user.LastName)"
  $department  = $user.Department
  $targetOU    = $roleMap[$department].OU
  $targetGroup = $roleMap[$department].Group

  if (-not $roleMap.ContainsKey($department)) {
    Write-Warning "Unknown department '$department' for $fullName — skipping"
    continue
  }

  New-ADUser `
    -Name              $fullName `
    -GivenName         $user.FirstName `
    -Surname           $user.LastName `
    -SamAccountName    $username `
    -UserPrincipalName "$username@lab.local" `
    -Title             $user.Title `
    -Department        $department `
    -Path              $targetOU `
    -AccountPassword   (ConvertTo-SecureString "Welcome@1234!" -AsPlainText -Force) `
    -ChangePasswordAtLogon $true `
    -Enabled           $true

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

| Group         | Rights                                                |
|---------------|-------------------------------------------------------|
| IT-Admins     | Domain Admins (full control)                          |
| IT-Staff      | Remote Desktop access, software installation rights   |
| HelpDesk      | Delegated: password reset, account unlock on Users OU |
| HR-Staff      | Access to HR file share only                          |
| Finance-Staff | Access to Finance file share only                     |
| Sales-Staff   | Access to Sales file share only                       |

### Verify Group Membership

```powershell
Get-ADGroupMember -Identity "IT-Staff"  | Select-Object Name, SamAccountName
Get-ADGroupMember -Identity "HelpDesk"  | Select-Object Name, SamAccountName
```

---

## 5. Group Policy Security Baselines

All GPOs were linked at the `OU=Corp` level unless otherwise noted and validated using `gpresult` and Event Viewer.

### Password & Account Lockout Policy

Applied via Default Domain Policy:

| Setting                     | Value      |
|-----------------------------|------------|
| Minimum password length     | 12 chars   |
| Password complexity         | Enabled    |
| Maximum password age        | 90 days    |
| Account lockout threshold   | 5 attempts |
| Lockout duration            | 30 minutes |
| Reset lockout counter after | 15 minutes |

```
GPO Path:
Computer Configuration > Policies > Windows Settings >
  Security Settings > Account Policies
```

### Workstation Hardening GPO

```
GPO Name: Workstation-Hardening
Linked to: OU=Workstations,OU=Computers,OU=Corp

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

  Default Security Level: Disallowed
  Additional Rules (Unrestricted):
    %WINDIR%\**            — allow Windows system binaries
    %PROGRAMFILES%\**      — allow installed applications
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

  - Audit Account Logon Events  — Success, Failure
  - Audit Account Management    — Success, Failure
  - Audit Logon Events          — Success, Failure
  - Audit Object Access         — Failure
  - Audit Policy Change         — Success
  - Audit Privilege Use         — Failure
  - Audit System Events         — Success, Failure
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

## 7. Patch Management with WSUS

Windows Server Update Services (WSUS) was installed on the domain controller to simulate enterprise patch lifecycle management — including computer group targeting, approval workflows, GPO-driven client configuration, and compliance reporting. This mirrors the patch management workflows used with SCCM/MECM and Intune in larger environments.

### Install the WSUS Role

```powershell
Install-WindowsFeature -Name UpdateServices, UpdateServices-WidDB, `
  UpdateServices-Services, UpdateServices-RSAT, `
  UpdateServices-API, UpdateServices-UI `
  -IncludeManagementTools
```

Run the post-install configuration to set the content and database paths:

```powershell
& "C:\Program Files\Update Services\Tools\WsusUtil.exe" `
  postinstall CONTENT_DIR=C:\WSUS
```

### Configure WSUS via PowerShell

```powershell
# Connect to the local WSUS server
$wsus = Get-WsusServer -Name "DC01" -PortNumber 8530

# Configure upstream sync from Microsoft Update
$wsusConfig = $wsus.GetConfiguration()
$wsusConfig.SyncFromMicrosoftUpdate = $true
$wsusConfig.Save()

# Set synchronization schedule — daily at 3:00 AM
$wsusSubscription = $wsus.GetSubscription()
$wsusSubscription.SynchronizeAutomatically = $true
$wsusSubscription.SynchronizeAutomaticallyTimeOfDay = (New-TimeSpan -Hours 3)
$wsusSubscription.NumberOfSynchronizationsPerDay = 1
$wsusSubscription.Save()

# Limit products and classifications to reduce initial sync time
Set-WsusProduct -WsusServer $wsus -Product "Windows 10", "Windows Server 2022"
Set-WsusClassification -WsusServer $wsus `
  -Classification "Critical Updates", "Security Updates", "Definition Updates"
```

### Create Computer Groups Mirroring OU Structure

```powershell
$wsus.CreateComputerTargetGroup("Workstations")
$wsus.CreateComputerTargetGroup("Servers")

# Verify groups were created
$wsus.GetComputerTargetGroups() | Select-Object Name
```

### Configure GPO to Point Clients at WSUS

Create a new GPO and link it to the Corp OU so all domain-joined machines check in to WSUS rather than Microsoft Update directly:

```
GPO Name: WSUS-Client-Configuration
Linked to: OU=Corp

Computer Configuration > Policies > Administrative Templates >
  Windows Components > Windows Update:

  - Specify intranet Microsoft update service location:
      Set the intranet update service for detecting updates:
        http://DC01:8530
      Set the intranet statistics server:
        http://DC01:8530

  - Configure Automatic Updates:
      Option: 4 — Auto download and schedule the install
      Scheduled install day: 0 (Every day)
      Scheduled install time: 03:00

  - Enable client-side targeting:
      Target group name for this computer: Workstations
```

Apply the GPO and force a check-in from the client:

```powershell
# On the Windows 10 client
gpupdate /force
wuauclt /detectnow
wuauclt /reportnow

# Alternatively on Windows 10/11
UsoClient StartScan
UsoClient RefreshSettings
```

### Configure Auto-Approval Rules

In the WSUS console, create an automatic approval rule for Critical and Security Updates targeting the Workstations group:

```
WSUS Console > Options > Automatic Approvals > New Rule:

  Step 1 — Properties:
    [x] When an update is in a specific classification
    [x] When an update is in a specific product

  Step 2 — Edit properties:
    Classification: Critical Updates, Security Updates
    Product: Windows 10

  Step 3 — Specify a name:
    Name: Auto-Approve Critical and Security — Workstations

  Step 4 — Apply rule
```

### Generate a Compliance Report

```powershell
# Get all computers registered in WSUS
$wsus.GetComputerTargets() | Select-Object FullDomainName, LastSyncTime,
  LastReportedStatusTime | Format-Table -AutoSize

# Get update compliance summary for the Workstations group
$group = $wsus.GetComputerTargetGroups() |
  Where-Object { $_.Name -eq "Workstations" }

$wsus.GetSummariesPerComputerTarget(
  (New-Object Microsoft.UpdateServices.Administration.UpdateScope),
  (New-Object Microsoft.UpdateServices.Administration.ComputerTargetScope)
) | Select-Object ComputerTarget, NotInstalledCount, InstalledCount,
    FailedCount, UnknownCount | Format-Table -AutoSize
```

### Troubleshooting Scenario — Client Not Appearing in WSUS Console

**Symptom:** The Windows 10 client has checked in via GPO but does not appear in the WSUS computer list.

**Diagnosis:**

```powershell
# On the client — check Windows Update event log for WSUS contact
Get-WinEvent -LogName "Microsoft-Windows-WindowsUpdateClient/Operational" |
  Where-Object { $_.Id -in 19, 20, 25, 31 } |
  Select-Object TimeCreated, Id, Message | Format-List

# Verify the WSUS server URL is set correctly in the registry
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" |
  Select-Object WUServer, WUStatusServer, TargetGroup

# Test HTTP connectivity to WSUS from the client
Invoke-WebRequest -Uri "http://DC01:8530/selfupdate/iuident.cab" -UseBasicParsing
```

**Root Cause:** The GPO had not yet applied because the client was still in the default `Computers` container rather than the `OU=Corp` OU where the WSUS GPO was linked.

**Resolution:**

```powershell
# On the DC — move the computer object to the correct OU
Get-ADComputer "WIN10-CLIENT" | Move-ADObject `
  -TargetPath "OU=Workstations,OU=Computers,OU=Corp,DC=lab,DC=local"

# On the client — force GPO refresh and WSUS re-registration
gpupdate /force
wuauclt /resetauthorization /detectnow
wuauclt /reportnow
```

---

## 8. Web Server Administration with IIS

Internet Information Services (IIS) was installed on the domain controller to simulate internal web hosting — a common sysadmin responsibility in environments that self-host intranet portals, internal apps, or management interfaces. An internal DNS record was created so the site is reachable by name across the domain.

### Install IIS Role

```powershell
Install-WindowsFeature -Name Web-Server, Web-Mgmt-Console, `
  Web-Asp-Net45, Web-Basic-Auth, Web-Windows-Auth, `
  Web-Log-Libraries, Web-Request-Monitor `
  -IncludeManagementTools
```

Verify IIS is running:

```powershell
Get-Service W3SVC
Start-Service W3SVC
Set-Service W3SVC -StartupType Automatic
```

### Create an Internal DNS Record

Add a DNS A record so `intranet.lab.local` resolves to the DC:

```powershell
Add-DnsServerResourceRecordA `
  -ZoneName "lab.local" `
  -Name "intranet" `
  -IPv4Address "192.168.56.10"

# Verify resolution from the domain-joined client
Resolve-DnsName intranet.lab.local
```

### Deploy an Internal Site

Create the site directory and a placeholder HTML page:

```powershell
New-Item -ItemType Directory -Path "C:\inetpub\intranet" -Force

@"
<!DOCTYPE html>
<html>
<head><title>Lab Intranet</title></head>
<body>
  <h1>Lab Intranet Portal</h1>
  <p>Internal site hosted on DC01 — lab.local</p>
</body>
</html>
"@ | Set-Content "C:\inetpub\intranet\index.html"
```

Create a new IIS site bound to the internal DNS name on port 80:

```powershell
Import-Module WebAdministration

New-Website `
  -Name "Intranet" `
  -PhysicalPath "C:\inetpub\intranet" `
  -Port 80 `
  -HostHeader "intranet.lab.local" `
  -ApplicationPool "DefaultAppPool"

Start-Website -Name "Intranet"
Get-Website -Name "Intranet"
```

Test the site from the DC:

```powershell
Invoke-WebRequest -Uri "http://intranet.lab.local" -UseBasicParsing |
  Select-Object StatusCode, StatusDescription
```

### Configure Application Pool Identity

Set the application pool to run under `NetworkService` and configure a proper identity:

```powershell
Set-ItemProperty "IIS:\AppPools\DefaultAppPool" `
  -Name processModel.userName -Value "NetworkService"
Set-ItemProperty "IIS:\AppPools\DefaultAppPool" `
  -Name processModel.password -Value ""
Set-ItemProperty "IIS:\AppPools\DefaultAppPool" `
  -Name processModel.identityType -Value 2

Restart-WebAppPool -Name "DefaultAppPool"
```

### Add HTTPS with a Self-Signed Certificate

Create a self-signed certificate and bind it to the site on port 443:

```powershell
# Create a self-signed certificate valid for 1 year
$cert = New-SelfSignedCertificate `
  -DnsName "intranet.lab.local" `
  -CertStoreLocation "Cert:\LocalMachine\My" `
  -NotAfter (Get-Date).AddYears(1)

# Add HTTPS binding to the Intranet site
New-WebBinding `
  -Name "Intranet" `
  -Protocol "https" `
  -Port 443 `
  -HostHeader "intranet.lab.local" `
  -SslFlags 0

# Bind the certificate to the HTTPS listener
$binding = Get-WebBinding -Name "Intranet" -Protocol "https"
$binding.AddSslCertificate($cert.Thumbprint, "My")

# Verify both bindings are present
Get-WebBinding -Name "Intranet" | Select-Object protocol, bindingInformation
```

> **Note:** The self-signed certificate will trigger a browser trust warning since it is not issued by a trusted CA. Section 9 replaces this certificate with one issued by the lab's internal ADCS Enterprise CA, eliminating the warning on domain-joined machines.

### Configure IIS Logging

Verify IIS logging is enabled and confirm the log path:

```powershell
Get-WebConfigurationProperty `
  -Filter "system.applicationHost/sites/site[@name='Intranet']/logFile" `
  -PSPath "IIS:\" `
  -Name directory

# View recent IIS access logs
Get-ChildItem "C:\inetpub\logs\LogFiles\W3SVC*" |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1 |
  Get-Content -Tail 20
```

### Troubleshooting Scenario — Site Returns 503 Service Unavailable

**Symptom:** Browsing to `http://intranet.lab.local` returns a 503 error.

**Diagnosis:**

```powershell
# Check application pool state
Get-WebConfigurationProperty `
  -Filter "system.applicationHost/applicationPools/add[@name='DefaultAppPool']" `
  -PSPath "IIS:\" -Name state

# Check IIS event log for application pool failure
Get-WinEvent -LogName "System" |
  Where-Object { $_.ProviderName -eq "WAS" } |
  Select-Object TimeCreated, Id, Message |
  Format-List

# Also check the Application log
Get-WinEvent -LogName "Application" |
  Where-Object { $_.LevelDisplayName -eq "Error" } |
  Select-Object -First 10 TimeCreated, ProviderName, Message |
  Format-List
```

Look for Event ID `5002` (application pool disabled due to rapid failures) or `5059` (rapid fail protection triggered).

**Root Cause:** The application pool stopped after exceeding the rapid-fail protection threshold — typically caused by a misconfigured site path or a worker process crash.

**Resolution:**

```powershell
# Verify the physical path exists and is accessible
Test-Path "C:\inetpub\intranet"

# Restart the application pool
Start-WebAppPool -Name "DefaultAppPool"

# Confirm the pool is now running
Get-WebConfigurationProperty `
  -Filter "system.applicationHost/applicationPools/add[@name='DefaultAppPool']" `
  -PSPath "IIS:\" -Name state

# Re-test the site
Invoke-WebRequest -Uri "http://intranet.lab.local" -UseBasicParsing |
  Select-Object StatusCode
```

---

## 9. Certificate Services with ADCS

Active Directory Certificate Services (ADCS) was installed as an Enterprise Certification Authority, enabling the lab domain to issue trusted certificates to domain-joined machines and services. The CA-issued certificate from this section replaces the self-signed certificate on the IIS intranet site, eliminating browser trust warnings on domain-joined clients.

### Install ADCS Role

```powershell
Install-WindowsFeature -Name ADCS-Cert-Authority, ADCS-Web-Enrollment `
  -IncludeManagementTools
```

### Configure the Enterprise CA

```powershell
Install-AdcsCertificationAuthority `
  -CAType EnterpriseRootCA `
  -CACommonName "Lab-Root-CA" `
  -KeyLength 2048 `
  -HashAlgorithmName SHA256 `
  -ValidityPeriod Years `
  -ValidityPeriodUnits 5 `
  -Force
```

> **Enterprise CA vs Standalone CA:** An Enterprise CA integrates with Active Directory, enabling auto-enrollment via Group Policy and domain-aware certificate templates. A Standalone CA operates independently of AD and requires manual request and approval workflows. For domain-joined environments, Enterprise CA is the standard choice.

Verify the CA is running:

```powershell
Get-Service CertSvc
Get-CACertificate | Select-Object Subject, NotBefore, NotAfter
```

### Verify Domain Clients Trust the CA Automatically

Because this is an Enterprise CA integrated with AD, domain-joined machines automatically receive the CA certificate in their Trusted Root Certification Authorities store via Group Policy. Verify on the Windows 10 client:

```powershell
# Run on the domain-joined Windows 10 client
Get-ChildItem Cert:\LocalMachine\Root |
  Where-Object { $_.Subject -like "*Lab-Root-CA*" } |
  Select-Object Subject, Thumbprint, NotAfter
```

### Issue a Certificate for the IIS Intranet Site

On the DC, request a certificate from the Enterprise CA for `intranet.lab.local` using the Web Server template:

```powershell
# Request a certificate using the WebServer template
$cert = Get-Certificate `
  -Template "WebServer" `
  -DnsName "intranet.lab.local" `
  -CertStoreLocation "Cert:\LocalMachine\My"

$cert.Certificate | Select-Object Subject, Thumbprint, NotAfter
```

Bind the CA-issued certificate to the IIS HTTPS listener, replacing the self-signed certificate:

```powershell
# Remove the old self-signed binding
Remove-WebBinding -Name "Intranet" -Protocol "https" -Port 443

# Add a new HTTPS binding
New-WebBinding `
  -Name "Intranet" `
  -Protocol "https" `
  -Port 443 `
  -HostHeader "intranet.lab.local" `
  -SslFlags 0

# Bind the new CA-issued certificate
$thumbprint = $cert.Certificate.Thumbprint
$binding = Get-WebBinding -Name "Intranet" -Protocol "https"
$binding.AddSslCertificate($thumbprint, "My")

# Verify
Get-WebBinding -Name "Intranet" | Select-Object protocol, bindingInformation
```

Domain-joined clients browsing to `https://intranet.lab.local` will now trust the certificate without a warning, since they received the CA root via Group Policy.

### Practice Certificate Renewal

```powershell
# View the expiration date of the current certificate
Get-ChildItem Cert:\LocalMachine\My |
  Where-Object { $_.Subject -like "*intranet*" } |
  Select-Object Subject, Thumbprint, NotAfter

# Renew the certificate (re-request from CA using same template)
$renewed = Get-Certificate `
  -Template "WebServer" `
  -DnsName "intranet.lab.local" `
  -CertStoreLocation "Cert:\LocalMachine\My" `
  -RenewExpired

$renewed.Certificate | Select-Object Subject, Thumbprint, NotAfter
```

### Practice Certificate Revocation

```powershell
# Revoke a certificate by serial number (obtain serial from cert properties)
$serial = (Get-ChildItem Cert:\LocalMachine\My |
  Where-Object { $_.Subject -like "*intranet*" }).SerialNumber

Revoke-CACertificate -SerialNumber $serial -Reason "Superseded"

# Publish an updated CRL immediately
Invoke-Command -ScriptBlock { certutil -crl }

# Verify the certificate appears in the revocation list
Get-CARevokedCert | Select-Object SerialNumber, RevocationDate, RevocationReason
```

---

## 10. Microsoft Entra ID

Microsoft Entra ID was configured as a standalone cloud identity extension to explore hybrid identity concepts alongside the on-premises AD environment.

### Create Users and Groups

Users and groups were created in the Entra ID portal and via PowerShell using the Microsoft Graph module:

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser
Connect-MgGraph -Scopes "User.ReadWrite.All", "Group.ReadWrite.All"

New-MgUser `
  -DisplayName "Test User" `
  -UserPrincipalName "testuser@<tenant>.onmicrosoft.com" `
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

Verified by signing in as the test user and confirming the MFA prompt via the Microsoft Authenticator app.

### Conditional Access Policy

A Conditional Access policy was configured to require MFA for all users signing in from outside a trusted location:

```
Azure Portal > Microsoft Entra ID > Security > Conditional Access > New Policy:

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

> This was implemented as a standalone extension using a free Microsoft Entra ID tenant to understand cloud identity models alongside on-premises Active Directory, rather than a live sync with the lab domain.

---

## 11. Cloud Monitoring with Azure Monitor & Log Analytics

A Log Analytics workspace was created in the free Azure subscription to extend monitoring beyond the on-premises lab into the cloud identity layer. Entra ID sign-in and audit logs are collected, queried with KQL (Kusto Query Language), and surfaced through alert rules — simulating the monitoring workflows used in hybrid environments running Azure Monitor alongside on-premises tools like SCOM.

### Create a Log Analytics Workspace

In the Azure portal:

```
Azure Portal > Log Analytics workspaces > Create:

  Subscription: <your subscription>
  Resource group: lab-monitoring-rg (create new)
  Name: lab-log-analytics
  Region: East US (or closest region)

Click: Review + Create > Create
```

### Connect Entra ID Logs to the Workspace

Route Entra ID sign-in and audit logs into the Log Analytics workspace:

```
Azure Portal > Microsoft Entra ID > Diagnostic settings > Add diagnostic setting:

  Name: EntraID-to-LogAnalytics
  Logs:
    [x] AuditLogs
    [x] SignInLogs
    [x] NonInteractiveUserSignInLogs
    [x] RiskyUsers
  Destination:
    [x] Send to Log Analytics workspace
    Workspace: lab-log-analytics

Click: Save
```

> Allow 5–15 minutes for logs to begin flowing into the workspace after saving.

### Query Logs with KQL

Open the Log Analytics workspace and navigate to **Logs** to run KQL queries against the collected data.

**Failed sign-in attempts in the last 24 hours:**

```kql
SignInLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"
| project TimeGenerated, UserPrincipalName, ResultType, ResultDescription,
    IPAddress, Location, AppDisplayName, ConditionalAccessStatus
| order by TimeGenerated desc
```

**Conditional Access policy failures:**

```kql
SignInLogs
| where TimeGenerated > ago(24h)
| where ConditionalAccessStatus == "failure"
| project TimeGenerated, UserPrincipalName, IPAddress,
    AppDisplayName, ConditionalAccessPolicies
| order by TimeGenerated desc
```

**User account changes from audit logs:**

```kql
AuditLogs
| where TimeGenerated > ago(7d)
| where Category == "UserManagement"
| project TimeGenerated, OperationName, InitiatedBy,
    TargetResources, Result
| order by TimeGenerated desc
```

**Count failed sign-ins per user over the past hour (brute-force detection pattern):**

```kql
SignInLogs
| where TimeGenerated > ago(1h)
| where ResultType != "0"
| summarize FailedAttempts = count() by UserPrincipalName, IPAddress
| where FailedAttempts > 3
| order by FailedAttempts desc
```

### Create an Alert Rule

Configure an alert that fires when any user exceeds 5 failed sign-in attempts within 10 minutes — a basic brute-force detection pattern:

```
Azure Portal > Log Analytics workspaces > lab-log-analytics >
  Alerts > New alert rule:

  Scope: lab-log-analytics workspace

  Condition:
    Signal: Custom log search
    Query:
      SignInLogs
      | where TimeGenerated > ago(10m)
      | where ResultType != "0"
      | summarize FailedAttempts = count() by UserPrincipalName
      | where FailedAttempts > 5

    Alert logic:
      Operator: Greater than
      Threshold value: 0
      Evaluation frequency: Every 5 minutes
      Lookback period: 10 minutes

  Actions:
    Create action group > Email notification to lab admin address

  Alert rule details:
    Name: Brute Force Detection — Failed Sign-Ins
    Severity: 2 — Warning

Click: Review + Create > Create
```

### Simulate an Investigation

Trigger several intentional failed sign-in attempts on the Entra ID test user, then run the KQL queries above to trace the events:

```
1. Attempt to sign in as testuser@<tenant>.onmicrosoft.com with a wrong password
   4-5 times from the Azure portal or https://myapps.microsoft.com

2. Wait 5 minutes, then run the failed sign-in query in Log Analytics

3. Identify the ResultType code (e.g., 50126 = invalid credentials),
   the source IP, and whether Conditional Access evaluated the attempt

4. Cross-reference with the AuditLogs query to see if any account
   changes occurred around the same timeframe
```

This workflow mirrors the investigation process used when responding to identity-related alerts in a SOC or hybrid infrastructure role.

---

## 12. Troubleshooting Scenarios

Real failure scenarios reproduced in the lab, documenting diagnosis steps and resolutions. All diagnosis involved a combination of Event Viewer and PowerShell, matching the workflow used in enterprise helpdesk and sysadmin roles.

---

### Scenario 1 — GPO Not Applying to Client

**Symptom:** A workstation hardening policy is not taking effect on the Windows 10 client after being linked to the Workstations OU.

**Diagnosis:**

```powershell
gpresult /r

Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational" |
  Where-Object { $_.LevelDisplayName -eq "Error" -or
                 $_.LevelDisplayName -eq "Warning" } |
  Select-Object TimeCreated, Id, Message | Format-List
```

Navigate in Event Viewer:

```
Event Viewer > Applications and Services Logs >
  Microsoft > Windows > GroupPolicy > Operational
```

Look for Event ID `7016` (policy processing error) or `1085` (policy failed to apply).

**Root Cause:** The computer object was still in the default `Computers` container, not in `OU=Workstations` where the GPO was linked.

**Resolution:**

```powershell
Get-ADComputer "WIN10-CLIENT" | Move-ADObject `
  -TargetPath "OU=Workstations,OU=Computers,OU=Corp,DC=lab,DC=local"

gpupdate /force
gpresult /r
```

---

### Scenario 2 — DNS Misconfiguration Breaks Domain Join

**Symptom:** Attempting to join a workstation to `lab.local` fails with "The specified domain either does not exist or could not be contacted."

**Diagnosis:**

```powershell
Resolve-DnsName lab.local
nslookup lab.local
Get-DnsClientServerAddress
Test-NetConnection -ComputerName 192.168.56.10 -Port 389
```

Check the DNS Server event log on the DC:

```
Event Viewer > Windows Logs > DNS Server
Event ID 4013 — DNS server waiting for AD
Event ID 4015 — DNS server critical error
```

**Root Cause:** The Windows 10 client's DNS was pointing to the VirtualBox NAT gateway instead of the DC at `192.168.56.10`.

**Resolution:**

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 192.168.56.10

ipconfig /flushdns
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

---

### Scenario 3 — Account Lockout

**Symptom:** A user cannot log in and reports their password is correct.

**Diagnosis:**

```powershell
Get-ADUser "jcarter" -Properties LockedOut, BadLogonCount,
  BadPasswordTime, LastLogonDate |
  Select-Object Name, LockedOut, BadLogonCount, BadPasswordTime

Get-WinEvent -FilterHashtable @{
  LogName = 'Security'
  Id      = 4740
} | Select-Object TimeCreated, Message | Format-List
```

Event ID `4740` will show the caller computer name — the machine triggering repeated failed authentications from a stale saved credential.

**Root Cause:** A stale cached credential on a shared workstation was repeatedly authenticating with an old password after a password change.

**Resolution:**

```powershell
Unlock-ADAccount -Identity "jcarter"

Get-ADUser "jcarter" -Properties LockedOut |
  Select-Object Name, LockedOut
```

Clear the stale credential from the source machine:

```
Control Panel > Credential Manager > Windows Credentials >
  Remove the cached lab.local entry
```

---

### Scenario 4 — Broken File Share Permissions

**Symptom:** A user in the `Finance-Staff` group cannot access the Finance file share despite being a group member.

**Diagnosis:**

```powershell
Get-ADUser "mlopez" -Properties MemberOf |
  Select-Object -ExpandProperty MemberOf

Get-Acl "C:\Shares\Finance" | Format-List
Get-SmbShareAccess -Name "Finance"
```

Check for access denied events:

```
Event Viewer > Windows Logs > Security
Event ID 4663 — Object access attempt
Event ID 5145 — Network share access check
```

**Root Cause:** The NTFS permissions granted access to `Finance-Staff`, but the SMB share permission was locked to `IT-Admins` only, blocking all other users at the share layer before NTFS permissions were evaluated.

**Resolution:**

```powershell
Grant-SmbShareAccess -Name "Finance" -AccountName "LAB\Finance-Staff" `
  -AccessRight Change -Force

Get-SmbShareAccess -Name "Finance"
Get-Acl "C:\Shares\Finance" | Select-Object -ExpandProperty Access
```

---

### Scenario 5 — Domain Join Failure Due to Duplicate Computer Object

**Symptom:** Rejoining a reimaged workstation to the domain fails with "The account already exists."

**Diagnosis:**

```powershell
Get-ADComputer "WIN10-CLIENT"

Get-ADComputer "WIN10-CLIENT" -Properties LastLogonDate |
  Select-Object Name, LastLogonDate
```

Check Event Viewer on the DC:

```
Event Viewer > Windows Logs > System
Event ID 5722 — Session setup from computer WIN10-CLIENT failed
```

**Root Cause:** A stale computer account from the previous installation remained in AD.

**Resolution:**

```powershell
Remove-ADComputer -Identity "WIN10-CLIENT" -Confirm:$false
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

---

### Scenario 6 — WSUS Client Not Appearing in Console

Documented in [Section 7](#7-patch-management-with-wsus).

---

### Scenario 7 — IIS Site Returns 503 Service Unavailable

Documented in [Section 8](#8-web-server-administration-with-iis).

---

## Tools Used

| Tool                        | Purpose                                                              |
|-----------------------------|----------------------------------------------------------------------|
| Active Directory Users      | GUI-based user, group, and OU management                             |
| and Computers (ADUC)        |                                                                      |
| PowerShell AD module        | Bulk provisioning, querying, and automation                          |
| Group Policy Management     | GPO creation, linking, and reporting                                 |
| Console (GPMC)              |                                                                      |
| gpresult                    | Verify applied GPOs on client and computer scope                     |
| gpupdate                    | Force policy refresh                                                 |
| Event Viewer                | Diagnose GPO failures, logon events, lockouts, DNS errors,           |
|                             | IIS application pool failures, and access denials                    |
| Get-WinEvent                | PowerShell-based event log querying                                  |
| Get-ADUser / Get-ADGroup    | AD object inspection and membership verification                     |
| Test-NetConnection          | Port and connectivity testing                                        |
| Resolve-DnsName             | DNS resolution testing                                               |
| Get-Acl / Get-SmbShare      | NTFS and share permission inspection                                 |
| WSUS Console                | Patch approval, computer group management, and compliance reporting  |
| WsusUtil.exe                | WSUS post-install configuration and content management               |
| IIS Manager                 | Web site and application pool administration                         |
| WebAdministration module    | PowerShell-based IIS configuration and binding management            |
| ADCS / Certification        | Enterprise CA management, certificate issuance and revocation        |
| Authority Console           |                                                                      |
| certutil                    | Certificate operations, CRL publishing, and CA diagnostics           |
| Microsoft Entra ID          | Cloud identity, MFA, and Conditional Access                          |
| Microsoft Graph PS          | Cloud user and group management via PowerShell                       |
| Azure Monitor               | Cloud-based infrastructure and identity monitoring                   |
| Log Analytics / KQL         | Log aggregation, querying, and alert rule configuration              |

---

[![Email](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:kennymiranda000@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/kenneth-miranda-xyz)
