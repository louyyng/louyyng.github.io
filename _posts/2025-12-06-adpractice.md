---
title: "Home Lab Setup For Practicing AD Attack"
date: 2025-12-06
categories:
  - blog
tags:
  - AD
author_profile: true
read_time: true
comments: false
share: true
related: true
header:
  teaser: /images/profile.png
---

# Home Lab Setup For Practicing AD Attack
### Initial
+ Windows Server 2022 x1
+ Windows 10 x1
+ Attack Machine x1 

# Simple DC Setup:
1) Change the name of DC.
![1](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/1.png)

2) Choose "Add Roles and Features Wizard"
![2](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/2.png)

3) Select "Role-based or feature-based installation:
![3](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/3.png)

4) Click "Add Features"
![4](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/4.png)

5) Install!
![5](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/5.png)

6) After installation, select "Promote this server to a domain controller"
![6](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/6.png)

7) Choose "Add a new forest" and insert Root Domain Name e.g "adpractice.local"
![7](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/7.png)

8) Set password and "Next" "Next" "Next" until you can "Install"
![Screenshot 2025-12-04 at 11.07.06 AM](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/8.png)

9) After installation, the DC will be rebooted and you will see the username changed to DOMAIN\User
![9](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/9.png)

# Windows 10/11   
Since I am using local network, I changed DNS to my DC IP and also modify hostable.  
1) Rename PC and add it to Domain
![10](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/10.png)

2) After OK -> the Welcome msg will prompt
![11](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/11.png)

3) Add new Groups
![12](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/12.png)

4) Add few users  
Add:  
2xDomain Admin  
2xNormal User  

# Register Microsoft Defender  
1) You can register trial of Microsoft defender first.   
2) Download onboarding package in "System" -> "Setting" -> "Endpoints"  
![13](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/13.png)
3) After running the bat script, try to download Mimikatz.  
4) We can find the alert on the dashboard.  
![14](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/14.png)

# Creating Service Principal Name (SPN)
If we need to practice Kerberoasting, we need a service account.  
1) Create another user named SQLService/Others.  
2) Run the following command to associate an SPN with that user:  
```
setspn -a myDomainController/SQLService.adpractice.local:60111 adpractice\SQLService
```
![15](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/15.png)

```
setspn -T adpractice.local -Q */*
```
![16](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/adpractice/16.png)
