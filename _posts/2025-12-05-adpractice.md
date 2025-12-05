# Home Lab Setup For Practicing AD Attack

+ Windows Server 2022 x1
+ Windows 10 x2
+ Attack Machine x1 

Simple DC Setup:
1) Change the name of DC.
![image](https://hackmd.io/_uploads/ByaHv_Rb-l.png)

2) Choose "Add Roles and Features Wizard"
![Screenshot 2025-12-04 at 10.53.32 AM](https://hackmd.io/_uploads/ryd2P_AWZx.png)

3) Select "Role-based or feature-based installation:
![Screenshot 2025-12-04 at 10.53.49 AM](https://hackmd.io/_uploads/rkqpPd0-bl.png)

4) Click "Add Features"
![Screenshot 2025-12-04 at 10.54.46 AM](https://hackmd.io/_uploads/HkQWd_R-We.png)

5) Install!
![Screenshot 2025-12-04 at 10.55.37 AM](https://hackmd.io/_uploads/SJI4Ou0WZx.png)

6) After installation, select "Promote this server to a domain controller"
![Screenshot 2025-12-04 at 10.58.42 AM](https://hackmd.io/_uploads/rJkgFdCZZx.png)

8) Choose "Add a new forest" and insert Root Domain Name e.g "adpractice.local"
![Screenshot 2025-12-04 at 10.59.27 AM](https://hackmd.io/_uploads/HyqfFOAWWl.png)

9) Set password and "Next" "Next" "Next" until you can "Install"
![Screenshot 2025-12-04 at 11.07.06 AM](https://hackmd.io/_uploads/HkwJo_Rb-l.png)

10) After installation, the DC will be rebooted and you will see the username changed to DOMAIN\User
![Screenshot 2025-12-04 at 11.14.35 AM](https://hackmd.io/_uploads/rkts2dAZWe.png)

In Windows PC:
Since I am using local network, I changed DNS to my DC IP and also modify hostable.
![Screenshot 2025-12-04 at 12.04.06 PM](https://hackmd.io/_uploads/SkXS_YCWWl.png)

2) After OK -> the Welcome msg will prompt
![Screenshot 2025-12-04 at 12.04.52 PM](https://hackmd.io/_uploads/Bkg_OtRbbx.png)

3) Add new Groups
![Screenshot 2025-12-04 at 1.30.18 PM](https://hackmd.io/_uploads/HyL_3qRbWe.png)

4) Add few users
Add:
2xDomain Admin
2xNormal User

# Register Microsoft Defender
1) You can register trial of Microsoft defender first. 
2) Download onboarding package in "System" -> "Setting" -> "Endpoints"
![Screenshot 2025-12-04 at 10.36.01 PM](https://hackmd.io/_uploads/rkJDnGJfbe.png)
3) After running the bat script, try to download Mimikatz. 
4) We can find the alert on the dashboard.
![Screenshot 2025-12-05 at 4.57.42 PM](https://hackmd.io/_uploads/HyM50fxM-l.png)

# Creating Service Principal Name (SPN)
If we need to practice Kerberoasting, we need a service account.
1) Create a user named SQLService.
2) Run the following command to associate an SPN with that user:
```
setspn -a myDomainController/SQLService.adpractice.local:60111 adpractice\SQLService
```
![Screenshot 2025-12-04 at 4.50.09 PM](https://hackmd.io/_uploads/B1JUiaCZbg.png)

```
setspn -T adpractice.local -Q */*
```
![Screenshot 2025-12-04 at 4.56.09 PM](https://hackmd.io/_uploads/SJHhn6RWWx.png)
