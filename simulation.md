# Simulation

## Overview / Objectives
- **Scenario:** Single end-to-end incident involving: SSH brute-forcing > successful compromise > post-compromise interaction.
- **Goals:**
  - Verify that JSON/Syslog logs reach Log Analytics / Sentinel.
  - Verify analytics rules/alerts work as expected.
  - Observe attacker behaviour through Cowrie session recordings and JSON logs.
  - Document timeline, evidence and lessons learned.
  
  > Note: - In this simulation i will be acting as both the threat actor and the defensive team. Offensive actions will start with a 🟥 while defensive actions will start with a 🟦 .
 
---

  ### 🟦 Verifying successful SSH Login alert
  - First we will verify our real ssh login creates an alert

  ![successfulssh1](/assets/images/sucssh1.png)
  ![successfulssh2](/assets/images/sucssh2.png)

  - Now whenever there is a successful ssh login it will create an informational incident and the ip associated with it. this allows for more transparency and a wider scope for suspicous activity.


---

### Brute-Forcing SSH
- In this simulation i will be acting as both the threat actor and the defensive team. Offensive actions will start with a 🟥 while defensive actions will start with a 🟦 
>**Disclaimer:** Only perform offensive actions on resources you own and not on any third party resources as it could be illegal.
> Note: I have purposefully made the honeypot password weak so it can be bruteforced but MAKE SECURE PASSWORDS. this shows how weak passwords can easily be exploited.

- 🟥 **Hydra** is a well known password cracker that can brute force various services such as SSH. My command uses a username and pairs it with a password from my wordlist to target ssh and keeps going till it finds the correct password.
 

![hydra1](/assets/images/hydra1.png)

- 🟦 Now if we go on incidents we can see that a **high** severity incident has been created due to the brute force attempts by hydra.

![bf](/assets/images/bf.png)

### Analysing traffic on workbooks
- Now we can see all the suspicous traffic clearly in my workbook
