# Simulation

## Overview / Objectives
- **Scenario:** Single end-to-end incident involving: SSH brute-forcing > successful compromise > post-compromise interaction.
- **Objectives:**
  - Verify that JSON/Syslog logs reach Log Analytics / Sentinel.
  - Verify that analytics rules & alerts trigger as expected.
  - Observe attacker behaviour through Cowrie session recordings and JSON logs.
  
  > Note: -  I act as both the **threat actor** and the **defender** in this simulation. Offensive steps are marked 🟥 and defensive steps 🟦.
 
---

  ### 🟦 Verifying the successful SSH Login alert
- First verify that a legitimate SSH login on the real system triggers the intended alert and creates an informational incident that includes the IP address and login metadata.
![successfulssh1](/assets/images/sucssh1.png)

- This provides visibility into **all** access attempts - not just failures.
- It allows for the detection of an **account compromise** such as a sucessful login from an unusual location or time.
- It can help analysts link events and give them a deeper understanding into an attack. For example:
  - A lone successful SSH login from an unusual location may mean a data leak which includes passwords.
  - A successful SSH login after many bruteforce attempts likely means a bruteforce attack and the successful discovery of the password.

---

### Brute-Forcing SSH
>**Disclaimer:** Only perform offensive actions on resources you own and not on any third party resources as it could be illegal.
- I have intentionally configured the honeypot to use a weak password to demonstrate how common passwords can be discovered.
- **DO NOT** deploy weak credentials in production - use strong passwords and multi-factor authentication.
- [A good article on strong password hygiene](https://www.cisa.gov/audiences/small-and-medium-businesses/secure-your-business/require-strong-passwords)  

- 🟥 I used **Hydra** (a well known brute-forcing tool) to try username/password combinations against the honeypot's SSH service. The command pairs a username with passwords from a wordlist and attempts SSH logins until a valid credential is found.
![hydra1](/assets/images/hydra1.jpg)

- 🟦 Now if we go to the Incidents tab on the Sentinel dashboard we can see that the brute-force attempts generated a high severity incident in Sentinel due to the alert we have previously made.
![bf](/assets/images/bf.png)

### 🟦 Analysing traffic on workbooks
- The workbook shows traffic in a clear, filterable format which we can edit to suit our needs using KQL.
![workbooksim](/assets/images/workbooksim.jpg)

- In the ingested JSON logs we can clearly see failed bruteforce attempts. The username and passwords are displayed in the form **[username/password]**
- From the logs we can see a successful login using the credentials **`[root/123]`** - this was the credential i configured for the honeypot.
- The logs also include **source IPs** which can be **geolocated** (assuming the attacker is not using VPNs/proxies). I have hidden some columns for privacy in the screenshots.

### Terminal session recordings
- Cowrie records attacker terminal sessions, producing a replayable recording of the commands the attacker executes inside the honeypot which is valuable for threat-intelligence & behavioural analysis.
- 🟥 Now if we log in to the honeypot and run some commands, it will be captured by Cowrie.
![terminalsim](/assets/images/terminalsim.png)

- These recorded sessions let defenders observe
  - Exected commands and their sequence
  - File/script upload
  - Privilege escalation attempts
  
- 🟦 On the defender side, you can play these recordings through your real SSH login.
![recording](/assets/images/recording.png)

- You may be thinking "why are there so many errors", this is because I havent properly configured the honeypot to use certain binaries or to have certain files therefore many commands wont work.
> Note: Cowrie stores recordings; screenshots are shown here for simplicity.

---

## Remediation

### Containment
- Block malicious IPs using a firewall.
- For efficency create an automation rule to block suspicous IPs and create tickets for analysis.
- Remove compromised credentials/accounts
### Hardening
- Enforce strong password policies and use MFA for extra verification.
- Restrict access to important ports via allow-lists if feasible.

---

### Improvements & Next Steps
- Integrate automated response playbooks (e.g automatically create a Sentinel playbook that blocks the IP and assigns it a person for analysis.
- Add threat-intel enrichment (e.g VirusTotal) to provide more context in incidents
- Create behavioural detections - e.g multiple usernames tried from one IP within X minutes, or successful login from an unusual location followed by malicious commands. 

---

## Final Thoughts
- This simulation is intentionally **simple** but practical: it demonstrates how weak credentials let an attacker gain access, how a honeypot paired with a SIEM produces high-value telemetry and actionable threat intelligence.
