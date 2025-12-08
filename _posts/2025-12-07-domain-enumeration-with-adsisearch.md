---
title: Domain Enumeration with adsisearch
date: 2025-12-06
categories:
  - blog
tags:
  - AD
author_profile: true
read_time: true
comments: false
share: false
related: true
header:
  teaser: /images/profile.png
---

```powershell
$searcher = [adsisearcher]""
```

---

### 1. User Enumeration
**Find All Domain Users**
Filters out computers and groups, showing only "Person" objects.
```powershell
$searcher.Filter = "(&(objectClass=user)(objectCategory=person))"
$searcher.FindAll().Properties.name
```

**Find Accounts with "Password Not Required"**
Often legacy accounts or misconfigurations.
```powershell
# UAC Bit 32 = PASSWD_NOTREQD
$searcher.Filter = "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))"
$searcher.FindAll() | Select-Object Path
```

**Find Accounts with "Password Never Expires"**
Common for service accounts.
```powershell
# UAC Bit 65536 = DONT_EXPIRE_PASSWORD
$searcher.Filter = "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=65536))"
$searcher.FindAll().Properties.name
```

**Find Accounts with Description Fields (Password Mining)**
Admins sometimes leave passwords or sensitive info in the description.
```powershell
$searcher.Filter = "(&(objectClass=user)(description=*))"
$searcher.FindAll() | ForEach-Object { 
    Write-Host "$($_.Properties.name) : $($_.Properties.description)" 
}
```

---

### 2. Attack Surface Enumeration (Roasting & Auth)
**Find Kerberoastable Accounts (SPNs)**
Finds users (not computers) that have a Service Principal Name set. These tickets can be cracked offline.
```powershell
$searcher.Filter = "(&(objectClass=user)(servicePrincipalName=*)(!(objectCategory=computer)))"
$searcher.FindAll() | Select-Object @{N='User';E={$_.Properties.name}}, @{N='SPN';E={$_.Properties.serviceprincipalname}}
```

**Find AS-REP Roastable Accounts (No Pre-Auth)**
Accounts that do not require Kerberos Pre-Authentication.
```powershell
# UAC Bit 4194304 = DONT_REQ_PREAUTH
$searcher.Filter = "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))"
$searcher.FindAll().Properties.name
```

**Find Users with Reversible Encryption Enabled**
Extremely insecure configuration where AD stores the password in a format easily reversed to cleartext.
```powershell
# UAC Bit 128 = ENCRYPTED_TEXT_PWD_ALLOWED
$searcher.Filter = "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=128))"
$searcher.FindAll().Properties.name
```

---

### 3. Privileged Account Hunting
**Find "Protected" Users (AdminCount)**
Users who are (or were) members of protected groups like Domain Admins usually have `adminCount=1`.
```powershell
$searcher.Filter = "(&(objectClass=user)(adminCount=1))"
$searcher.FindAll().Properties.name
```

**Find Domain Admins (Indirectly)**
It is easier to search for the group and list members than to filter users by `MemberOf` (which requires the full Distinguished Name).
```powershell
$searcher.Filter = "(sAMAccountName=Domain Admins)"
$result = $searcher.FindOne()
$result.Properties.member # Lists the Distinguished Names of all admins
```

---

### 4. Computer & Infrastructure Enumeration
**Find All Windows Servers (Excluding Workstations)**
Filters by operating system string.
```powershell
$searcher.Filter = "(&(objectCategory=computer)(operatingSystem=*server*))"
$searcher.FindAll() | Select-Object @{N='Name';E={$_.Properties.name}}, @{N='OS';E={$_.Properties.operatingsystem}}
```

**Find Computers with Unconstrained Delegation**
If you compromise these, you can harvest TGTs of any user connecting to them (including Domain Admins).
```powershell
# UAC Bit 524288 = TRUSTED_FOR_DELEGATION
$searcher.Filter = "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))"
$searcher.FindAll().Properties.name
```

**Find Computers with Constrained Delegation**
```powershell
# UAC Bit 16777216 = TRUSTED_TO_AUTH_FOR_DELEGATION
$searcher.Filter = "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=16777216))"
$searcher.FindAll().Properties.name
```

---

### 5. Domain Structure & Trusts
**Enumerate Domain Trusts**
Finds other domains this domain trusts.
```powershell
$searcher.Filter = "(objectClass=trustedDomain)"
$searcher.FindAll() | ForEach-Object {
    Write-Host "Trust Partner: $($_.Properties.name)"
    Write-Host "Direction: $($_.Properties.trustdirection)"
    Write-Host "Attributes: $($_.Properties.trustattributes)"
    Write-Host "----------------"
}
```

**Enumerate All Organizational Units (OUs)**
Useful for mapping the structure of the organization.
```powershell
$searcher.Filter = "(objectCategory=organizationalUnit)"
$searcher.FindAll().Properties.name
```

**Enumerate Group Policy Objects (GPOs)**
GPOs are stored as `groupPolicyContainer` objects.
```powershell
$searcher.Filter = "(objectCategory=groupPolicyContainer)"
$searcher.FindAll() | Select-Object @{N='GUID';E={$_.Properties.name}}, @{N='DisplayName';E={$_.Properties.displayname}}
```

### 6. Special Finds (Shares & SQL)
**Find Published File Shares**
Not commonly used in modern AD, but if present, they are listed as `volume` objects.
```powershell
$searcher.Filter = "(objectCategory=volume)"
$searcher.FindAll() | Select-Object @{N='Name';E={$_.Properties.name}}, @{N='UNC';E={$_.Properties.uncname}}
```

**Find SQL Server Instances (Service Registration)**
Looks for SPNs registered to MSSQL.
```powershell
$searcher.Filter = "(servicePrincipalName=MSSQL*)"
$searcher.FindAll() | Select-Object @{N='Account';E={$_.Properties.name}}, @{N='SPN';E={$_.Properties.serviceprincipalname}}
```

### CMD One-Liner format
If you are strictly in a CMD shell and cannot open a full PowerShell session, you can wrap any of the above like this:

**Example: Get all Users with 'pass' in description via CMD:**
```cmd
powershell -Command "([adsisearcher]'(&(objectClass=user)(description=*pass*))').FindAll() | % {$_.Properties.name + ' - ' + $_.Properties.description}"
```

---

### 7. Recursive Group Membership (The "Magic" OID)
One of the most powerful LDAP features is the **"In Chain"** matching rule (OID `1.2.840.113556.1.4.1941`). This allows you to find an object inside a group *even if it is nested 5 levels deep*, without writing a loop script.

**Find ALL members of "Domain Admins" (including nested groups)**
```powershell
$daDistinguishedName = (([adsisearcher]"(name=Domain Admins)").FindOne().Properties.distinguishedname)
$searcher.Filter = "(memberOf:1.2.840.113556.1.4.1941:=$daDistinguishedName)"
$searcher.FindAll().Properties.name
```

**Find ALL groups a specific user belongs to (Recursively)**
Change `UserToCheck` to the `sAMAccountName` of the target user.
```powershell
$userDN = (([adsisearcher]"(sAMAccountName=UserToCheck)").FindOne().Properties.distinguishedname)
$searcher.Filter = "(member:1.2.840.113556.1.4.1941:=$userDN)"
$searcher.FindAll().Properties.name
```

---

### 8. LAPS (Local Admin Password Solution)
If LAPS is installed, computer objects may contain the clear-text local admin password in a specific attribute (`ms-Mcs-AdmPwd`). By default, only Admins can read this, but sometimes permissions are misconfigured.

**Check if LAPS is present/used in the domain**
This looks for computers that have the LAPS password expiration timestamp set.
```powershell
$searcher.Filter = "(&(objectCategory=computer)(ms-Mcs-AdmPwdExpirationTime=*))"
$searcher.FindAll() | Select-Object @{N='PC';E={$_.Properties.name}}, @{N='PwdExpires';E={[datetime]::FromFileTime($_.Properties."ms-mcs-admpwdexpirationtime"[0])}}
```

**Attempt to read LAPS Cleartext Passwords**
If you run this and get results, you have local admin on those machines.
```powershell
$searcher.Filter = "(&(objectCategory=computer)(ms-Mcs-AdmPwd=*))"
$searcher.FindAll() | Select-Object @{N='PC';E={$_.Properties.name}}, @{N='Password';E={$_.Properties."ms-mcs-admpwd"}}
```

---

### 9. Fine-Grained Password Policies (PSO)
Standard domain password policies apply to everyone. Fine-Grained policies (PSOs) apply to specific groups (often Admins have *stricter* policies, or Service Accounts have *looser* ones).

**Enumerate Password Settings Objects**
```powershell
$searcher.Filter = "(objectClass=msDS-PasswordSettings)"
$searcher.FindAll() | Select-Object @{N='Name';E={$_.Properties.name}}, @{N='LockoutThresh';E={$_.Properties."msds-lockoutthreshold"}}, @{N='Precedence';E={$_.Properties."msds-psoapplies"}}
```

---

### 10. Exchange & Email Infrastructure
Exchange stores massive amounts of data in AD.

**Find Exchange Servers**
```powershell
$searcher.Filter = "(objectCategory=msExchExchangeServer)"
$searcher.FindAll().Properties.name
```

**Find Dynamic Distribution Groups**
These groups determine membership based on filters (like "All users in Sales"). Attackers inspect these to understand internal logical groupings.
```powershell
$searcher.Filter = "(objectClass=msExchDynamicDistributionList)"
$searcher.FindAll().Properties.name
```

---

### 11. Physical Network Topology
AD maps IP subnets to "Sites". This is critical for understanding where a specific IP is physically located (e.g., "HQ", "BranchOffice", "Cloud").

**List all Defined Subnets**
```powershell
$searcher.Filter = "(objectClass=subnet)"
$searcher.FindAll() | Select-Object @{N='Subnet';E={$_.Properties.name}}, @{N='Site';E={$_.Properties.siteobject}}
```

**List all Sites**
```powershell
$searcher.Filter = "(objectClass=site)"
$searcher.FindAll().Properties.name
```

---

### 12. Identifying "Stale" or "Ghost" Accounts
Cleaning up lists or finding hijackable accounts that no one is watching.

**Find Computers that haven't logged on in a long time (e.g., > 6 months)**
We use the `lastLogonTimestamp`. Note: This requires converting the date to LDAP "FileTime" format.
*Current FileTime roughly equals `133...`. A rough rule of thumb for 6 months ago is finding timestamps starting with lower numbers, but we can calculate it precisely.*

```powershell
# Calculate FileTime for 180 days ago
$limit = (Get-Date).AddDays(-180).ToFileTime()

$searcher.Filter = "(&(objectCategory=computer)(lastLogonTimestamp<=$limit))"
$searcher.FindAll() | Select-Object @{N='Name';E={$_.Properties.name}}, @{N='LastLogon';E={[datetime]::FromFileTime($_.Properties.lastlogontimestamp[0])}}
```

**Find Empty Groups**
Groups that have no members.
```powershell
$searcher.Filter = "(&(objectCategory=group)(!member=*))"
$searcher.FindAll().Properties.name
```

---

### 13. Advanced User Attributes (SID & Script Path)
**Find Users with Legacy Logon Scripts**
Logon scripts often contain hardcoded mapped drives or passwords.
```powershell
$searcher.Filter = "(&(objectCategory=person)(scriptPath=*))"
$searcher.FindAll() | Select-Object @{N='User';E={$_.Properties.name}}, @{N='Script';E={$_.Properties.scriptpath}}
```

**Find a User by their SID (Security Identifier)**
If you see a raw SID in an event log (e.g., `S-1-5-21-...-500`) and want to know who it is.
```powershell
$TargetSID = "S-1-5-21-YOUR-SID-HERE" 
$searcher.Filter = "(objectSid=$TargetSID)"
# Note: objectSid is binary, so we usually search by the string representation if the LDAP server supports it, 
# strictly speaking, standard LDAP queries on objectSid need binary hex.
# However, for simple resolving, we can just search for * everything and filter client side if the SID is unknown:
$searcher.Filter = "(objectCategory=person)"
$searcher.FindAll() | Where-Object { (new-object System.Security.Principal.SecurityIdentifier $_.Properties.objectSid[0],0).Value -eq $TargetSID }
```

### 14. Certificate Authority (PKI) Enumeration
If the domain uses AD CS (Active Directory Certificate Services), this can be a major attack vector (e.g., PetitPotam, ESC1).

**Find Certificate Authorities**
```powershell
$searcher.Filter = "(objectCategory=pKIEnrollmentService)"
$searcher.FindAll() | Select-Object @{N='CA Name';E={$_.Properties.name}}, @{N='DNS';E={$_.Properties.dnshostname}}
```

**Find Certificate Templates**
Look for templates with `msPKI-Certificate-Name-Flag` settings that allow users to supply their own Subject Alternative Name (SAN).
```powershell
$searcher.Filter = "(objectCategory=pKICertificateTemplate)"
$searcher.FindAll().Properties.name
```

---

### 15. AD-Integrated DNS Enumeration
Active Directory often acts as the DNS server. DNS records are stored as objects (`dnsNode`). You can dump the internal DNS zones without performing a network-based Zone Transfer (AXFR).

**Find all DNS Records (A Records, CNAMEs, etc.)**
This effectively maps the internal network, revealing servers that might not be joined to the domain but have DNS entries (like Linux appliances or Intranet sites).
```powershell
$searcher.Filter = "(objectClass=dnsNode)"
$searcher.FindAll() | Select-Object @{N='Hostname';E={$_.Properties.name}}, @{N='IP/Data';E={$_.Properties."dnsrecord" | ForEach-Object {[System.Text.Encoding]::ASCII.GetString($_)}}}
```
*(Note: The `dnsrecord` is binary data; simply casting to string reveals the IP address inside the garbage text.)*

---

### 16. Group Policy (GPO) Links
We previously found the GPO objects themselves. Now we need to see **where** they are applied (which OUs they are linked to).

**Find OUs and list their linked GPOs**
The `gPLink` attribute contains the path to the GPO.
```powershell
$searcher.Filter = "(&(objectCategory=organizationalUnit)(gPLink=*))"
$searcher.FindAll() | Select-Object @{N='OU Name';E={$_.Properties.name}}, @{N='Linked GPOs';E={$_.Properties.gplink}}
```

**Find Blocked Inheritance**
OUs can be set to block policies from above. This is done via the `gPOptions` attribute (Value `1` = Block Inheritance).
```powershell
$searcher.Filter = "(&(objectCategory=organizationalUnit)(gPOptions=1))"
$searcher.FindAll().Properties.name
```

---

### 17. User Home Directories & Profiles
This is high-value for lateral movement. If a user's home directory or roaming profile is stored on a file share, and you have write access to that share, you can compromise the user when they next log in.

**Find Users with Roaming Profiles**
```powershell
$searcher.Filter = "(&(objectCategory=person)(profilePath=*))"
$searcher.FindAll() | Select-Object @{N='User';E={$_.Properties.name}}, @{N='ProfilePath';E={$_.Properties.profilepath}}
```

**Find Users with Mapped Home Drives**
```powershell
$searcher.Filter = "(&(objectCategory=person)(homeDirectory=*))"
$searcher.FindAll() | Select-Object @{N='User';E={$_.Properties.name}}, @{N='HomeDrive';E={$_.Properties.homedirectory}}
```

---

### 18. Printer Enumeration
Printers are often overlooked. They are "Computer" objects or "PrintQueue" objects.
1.  **Vulnerabilities:** Old printers often have default creds or old firmware.
2.  **Privilege Escalation:** If you can force a Domain Controller to connect to a printer you control (with a custom driver), you can trigger authentication (PrintNightmare scenarios).

**Find Published Printers**
```powershell
$searcher.Filter = "(objectCategory=printQueue)"
$searcher.FindAll() | Select-Object @{N='Printer';E={$_.Properties.printername}}, @{N='Server';E={$_.Properties.servername}}, @{N='Driver';E={$_.Properties.drivername}}
```

---

### 19. Foreign Security Principals (FSPs)
When a user from a *Trusted Domain* (Domain B) is added to a group in the *Current Domain* (Domain A), AD creates an FSP object. Enumerating these helps you understand cross-forest access.

**Find Security Principals from other domains**
```powershell
$searcher.Filter = "(objectClass=foreignSecurityPrincipal)"
$searcher.FindAll().Properties.name
```
*(The "Name" usually contains the SID of the foreign user).*

---

### 20. Service Connection Points (SCPs)
Applications register SCPs in AD to let clients find them (e.g., Skype for Business, SCCM, Hyper-V Clusters).

**Find SCCM (System Center Configuration Manager)**
SCCM Management Points are prime targets. If you compromise SCCM, you control all workstations.
```powershell
$searcher.Filter = "(&(objectClass=mSSMSManagementPoint))"
$searcher.FindAll() | Select-Object @{N='SCCM Server';E={$_.Properties.dcn}}, @{N='SiteCode';E={$_.Properties."ms-sms-site-code"}}
```

**Find Cluster Names**
```powershell
$searcher.Filter = "(serviceClassName=MSClusterDiscovery)"
$searcher.FindAll().Properties.name
```

---

### 21. Critical System Objects
These are specific objects that control how AD functions.

**AdminSDHolder**
This object controls the permissions for protected groups (Domain Admins, etc.). If an attacker modifies the ACL on *this* object, they can reset their permissions automatically every 60 minutes, even if you delete their account from the Admin group.
```powershell
$searcher.Filter = "(cn=AdminSDHolder)"
$searcher.FindOne().Path
```

**Default Password Policy (Non-PSO)**
This is stored in the domain root attributes.
```powershell
$domainRoot = [adsi]"LDAP://rootDSE"
$defaultNC = $domainRoot.defaultNamingContext
$domainObj = [adsi]"LDAP://$defaultNC"
# Max Password Age
$domainObj.maxPwdAge
# Min Password Length
$domainObj.minPwdLength
```

---

### 22. ACL / Ownership Scanning (Advanced)
`[adsisearcher]` returns `SearchResult` objects. To check permissions (ACLs), you technically need to convert the result to a `DirectoryEntry` using `.GetDirectoryEntry()`.

**Find objects owned by a specific user**
If a regular user "owns" a Critical Group or Computer, they can modify it to give themselves access.
*Replace `MYDOMAIN\User` with the target.*

```powershell
# Note: This is slower because it iterates results
$targetOwner = "MYDOMAIN\jsmith"
$searcher.Filter = "(objectClass=group)" # Checking groups only for speed
$searcher.FindAll() | ForEach-Object {
    $entry = $_.GetDirectoryEntry()
    if ($entry.ObjectSecurity.Owner -eq $targetOwner) {
        Write-Host "VULN: $targetOwner owns $($entry.Name)"
    }
}
```

---

### 23. Operating System Analytics
Useful for finding End-of-Life (EOL) systems (Windows 7, Server 2008) which are likely vulnerable to EternalBlue.

**Group Computers by Operating System Version**
```powershell
$searcher.Filter = "(objectCategory=computer)"
$searcher.FindAll() | Group-Object {$_.Properties.operatingsystem} | Select-Object Count, Name
```

---

### Quick Reference: Common Bitmasks
When filtering `userAccountControl` (UAC), use these numbers in your filters.
*Example: `(userAccountControl:1.2.840.113556.1.4.803:=NUMBER)`*

*   **512**: Normal Account
*   **514**: Disabled Account
*   **4096**: Workstation/Server (Machine Account)
*   **8192**: Domain Controller
*   **65536**: Password Never Expires
*   **4194304**: Does not require Kerberos Pre-Auth (Roastable)
*   **524288**: Trusted for Delegation (Unconstrained - High Risk)
*   **16777216**: Trusted to Authenticate for Delegation (Constrained)

---

Here are enumeration techniques focusing on **Azure AD (Hybrid Identity)**, **Deleted Objects**, **Delegation Attacks**, and **Schema Extensions**.

### 24. Azure AD / Entra ID Synchronization
In hybrid environments, an on-premise service account synchronizes AD to the cloud. Compromising this account (or the server it runs on) allows an attacker to manipulate the cloud environment (e.g., "Password Hash Sync" extraction).

**Find the Azure AD Connector Account**
This account usually starts with `MSOL_` (older) or `AAD_` (newer) and has a distinct description.
```powershell
$searcher.Filter = "(name=MSOL_*)"
$searcher.FindAll() | Select-Object @{N='Account';E={$_.Properties.name}}, @{N='Description';E={$_.Properties.description}}
```

**Find Users Synced to the Cloud**
Azure AD Connect uses the `ms-DS-ConsistencyGuid` as the "Immutable ID" (Source Anchor) to link on-prem users to cloud users. If this is present, the user is likely synced.
```powershell
$searcher.Filter = "(&(objectClass=user)(ms-DS-ConsistencyGuid=*))"
$searcher.FindAll().Properties.name
```

---

### 25. Recovering Deleted Objects (Tombstones)
When an object is deleted, it isn't removed immediately; it is hidden in a "Deleted Objects" container for 180 days (default). You can find these to see what old accounts existed or to recover them.

**Important:** You must set the `.Tombstone` property to `$true` to see these.

```powershell
$searcher.Tombstone = $true
$searcher.Filter = "(isDeleted=TRUE)"
$searcher.FindAll() | Select-Object @{N='Name';E={$_.Properties.name}}, @{N='LastKnownParent';E={$_.Properties.lastknownparent}}
```
*Note: The `name` will look like `UserName\0ADEL:GUID...`.*

---

### 26. Windows LAPS (New vs. Legacy)
We covered Legacy LAPS (`ms-Mcs-AdmPwd`) previously. The **New Windows LAPS** (built into Server 2019/2022+) uses different attributes.

**Check for Windows LAPS Usage**
```powershell
# Windows LAPS uses msLAPS-PasswordExpirationTime
$searcher.Filter = "(&(objectCategory=computer)(msLAPS-PasswordExpirationTime=*))"
$searcher.FindAll().Properties.name
```

**Dump Windows LAPS Passwords (Cleartext)**
*Requires Decrypt rights or Domain Admin.*
```powershell
$searcher.Filter = "(&(objectCategory=computer)(msLAPS-Password=*))"
$searcher.FindAll() | Select-Object @{N='Computer';E={$_.Properties.name}}, @{N='JSON_Password';E={$_.Properties."mslaps-password"}}
```
*Note: The password is stored as a small JSON string inside the attribute.*

---

### 27. Resource-Based Constrained Delegation (RBCD)
Standard delegation uses `msDS-AllowedToDelegateTo`. **RBCD** is more dangerous because it is configured on the *target* computer, not the source. It allows an attacker to say "I trust Computer A to impersonate anyone to me."

The attribute `msDS-AllowedToActOnBehalfOfOtherIdentity` stores a binary Security Descriptor (SD), making it hard to read. This script decodes it to show **who** is allowed to hack the target.

```powershell
$searcher.Filter = "(msDS-AllowedToActOnBehalfOfOtherIdentity=*)"
$searcher.FindAll() | ForEach-Object {
    $targetName = $_.Properties.name[0]
    $rawBytes   = $_.Properties["msds-allowedtoactonbehalfofotheridentity"][0]
    
    # Decode the Security Descriptor
    $SD = New-Object System.Security.AccessControl.RawSecurityDescriptor ($rawBytes, 0)
    $SDDL = $SD.GetSddlForm("Access")

    [PSCustomObject]@{
        TargetComputer = $targetName
        SDDL_String    = $SDDL # Shows the SID of the trusted account
    }
}
```
*If you see a result, the account listed in the SDDL (look for the SID) can compromise the `TargetComputer`.*

---

### 28. Schema Extensions (Custom Applications)
Organizations often extend the AD Schema for third-party apps (SAP, Oracle, Cisco). These custom attributes often contain sensitive data because admins forget to secure them.

**Find Custom Schema Attributes**
We look into the `Schema` naming context for attributes that are NOT standard Microsoft ones (often checking for specific prefixes or lack of Microsoft origin).

```powershell
# Get Schema Path
$root = [adsi]"LDAP://rootDSE"
$schemaPath = $root.schemaNamingContext

# Search inside the Schema partition
$schemaSearcher = [adsisearcher]""
$schemaSearcher.SearchRoot = [adsi]"LDAP://$schemaPath"

# Find attributes that are NOT from Microsoft (OID usually starts with 1.2.840.113556 for MS)
# This is a rough filter for 'interesting' extensions
$schemaSearcher.Filter = "(&(objectClass=attributeSchema)(!attributeID=1.2.840.113556*))"
$schemaSearcher.FindAll() | Select-Object @{N='Attr';E={$_.Properties.lDAPDisplayName}}, @{N='OID';E={$_.Properties.attributeid}}
```

---

### 29. "Unconstrained" Delegation (The Dangerous Ones)
If a computer has `TRUSTED_FOR_DELEGATION` (Unconstrained), and a Domain Admin connects to it, their TGT (Ticket Granting Ticket) is left in memory. You can harvest it and become Domain Admin.

```powershell
# UAC Bit 524288 = TRUSTED_FOR_DELEGATION
# Exclude Domain Controllers (8192) because they always have this, we want MEMBER servers.
$searcher.Filter = "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288)(!userAccountControl:1.2.840.113556.1.4.803:=8192))"
$searcher.FindAll().Properties.name
```

### Summary of `[adsisearcher]` Properties
When using this tool, remember these key properties you can modify on the `$searcher` object before running `.FindAll()`:

1.  `$searcher.SearchRoot = [adsi]"LDAP://OU=Sales,DC=contoso,DC=com"` (Limit scope to one OU).
2.  `$searcher.Tombstone = $true` (Find deleted items).
3.  `$searcher.PageSize = 1000` (Required if you expect >1000 results, otherwise AD stops returning data).
4.  `$searcher.PropertiesToLoad.Add("description")` (Optimization: Only fetch specific fields to speed up queries).