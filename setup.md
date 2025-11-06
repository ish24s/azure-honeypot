# 🧩 Setup & Configuration


## Setup Goals
- Deploy a **Azure VM** as a secure, monitored host environment.
- Install & configure a Cowrie SSH honeypot to simulate an interactive **SSH** service.
- Forward system & honeypot logs to **Microsoft Sentinel** for detection & analysis.
- Build custom **KQL** queries, workbooks and alert rules for real-time attack detection & visualisation.

---

### Creating an Azure VM
- To host the honeypot, a **Virtual Machine (VM)** is required to act as the server environment.
- There are 2 main approaches to this: hosting a VM locally using virtualisation software **(VirtualBox, VMware)** or delpoying one through a **cloud provider** **(Azure, AWS, GCP)**
- Each method has its pros & cons, but I chose **Microsoft Azure** due to its **30-day free trial**, **generous credit allowance** and native integration with Microsoft Sentinel
> **Note:** AWS and GCP also give free trials
- Using cloud infrastructure also reflects real-world practices, giving hands-on experience in cloud security, resource management and security monitoring.

VM Configs:
> **Note:** All other sections were left as default

![vmb1](/assets/images/vmbasics1.png)

> **Note:** You can also use **SSH public-key authentication** however i used password authentication for simplification and allow for brute-force attack simulation.

![vmb2](/assets/images/vmbasics2.png)
![vmd](/assets/images/vmdisk.png)

---

### Installing Cowrie
- To start with you will need a honeypot environment - common choices include **Cowrie**, **T-Pot**, **Dionaea** etc.
- I used Cowrie because it focuses on SSH emulation, provides detailed logging, has extensive features and is lightweight.
- Installation steps were followed from the **[official Cowrie documentation](https://docs.cowrie.org/en/latest/INSTALL.html)**  
- My configuration file `cowrie.cfg` is included in this repository for reference.

### Configuring SSH on port 22
- By default Cowrie runs on **port 2222** but it can listen on the standard SSH port (22) to appear realistic. To free port 22 for Cowrie, move your real SSH daemon to an alternate port.
#### Steps:
1. Edit the system SSH config (`/etc/ssh/sshd_config`) and change the port e.g `Port 1111`
2. Restart the SSH service after changing port (`sudo systemctl restart sshd`)
3. Enable SSH in cowrie.cfg (already present in my config):

```ini
[ssh]
enabled = true
listen_endpoints = tcp:22:interface=0.0.0.0
```

4. Finally configure the **NSG (Network Security Group)** on the Azure VM to allow inbound traffic for both ports: the honeypot (22) and your real SSH port (e.g 1111)

![nsg](/assets/images/nsg.png)

> **Note:** Give the rule for the real SSH port a higher priority (lower priority number) so you can still connect to the VM for administration. You should have two separate inbound rules: one for the real SSH port and one for the honeypot (22).


### Configuring a fake filesystem
- Cowrie allows for a fake filesystem which mimics a real **Linux environment** for attackers to interact with. It enhances realism and helps capture attacker behaviour which can help with **Threat Intelligence**.
#### Steps:
1. First create a directory with your fake filesystem contents e.g
`sudo mkdir -p /opt/cowrie/custom_honeyfs/{etc,home,root,var/log,usr/bin,bin}`
2. Generate the filesystem pickle (adjust paths to your installation `bin/createfs -l /home/cowrie/cowrie/customhoneyfs/ -d 3 -o customfs.pickle`
3. Lastly point Cowrie to the generated file system in cowrie.cfg.
```ini
[shell]
filesystem = /home/cowrie/cowrie/var/lib/cowrie/customfs.pickle
```
> **Note:** Paths in your installation may differ so use the correct paths for your VM .
4. Add fake users and passwords to `/cowrie/etc/userdb.txt` to simulate realistic brute-force targets.

---

### Log Forwarding
- To monitor Cowrie and system events centrally, logs must be forwarded by **Azure Monitor**.
- We will collect 2 types of logs:
1. **Syslog** - system activity (including SSH)
2. **JSON** - detailed Cowrie session logs

- Change the settings below in `cowrie.cfg` to allow log collection.
```ini
[output_jsonlog]
enabled = true
logfile = ${honeypot:log_path}/cowrie.json
epoch_timestamp = false
```

```ini
[output_localsyslog]
enabled = true
facility = AUTH
format = text
```

#### Azure Configuration
1. Create a **Log Analytics Workspace (LAW)** in the same region and resource group as your VM.
2. Install the Azure Monitor Agent (AMA) on your VM.
> **Note:** There is no need to do anything as it will be installed automatically when you create a DCR. There are other ways to install AMA as seen [here](itor/agents/azure-monitor-agent-manage?tabs=azure-portal#installation-options).
3. Create both a **Data Collection Endpoint (DCE)** and **Data Collection Rule (DCR)** 
4. In the DCR, add **Linux Syslog** and **Custom JSON logs** as data sources
  - The JSON schema should match `cowrie.json` which is located in the Cowrie logs directory.
  - For Syslog, enable facilities **LOG_AUTH** and **LOG_AUTHPRIV** to capture authentication events.

![datasources](/assets/images/ds.png)
![syslog](/assets/images/syslog.png)
![json](/assets/images/custjson.png)

#### Verification
- Test data ingestion in the LAW using `Syslog | take 10`
- If logs appear, the setup works.
- To ingest Cowrie JSON logs, first create a custom table in the LAW and define its schema using a sample JSON entry from `cowrie.json`.
- To test JSON logs use the same command as above but replace `Syslog` with your table name e.g `JSON_CL | take 10`
> **Note:** tables in LAW end in _CL

---

### Microsoft Sentinel
- Microsoft Sentinel provides **SIEM & SOAR** capabilities for alerting, analytics & visualisation.
> Note: While Microsoft Sentinel is a cloud-based SIEM, there are alternative free or open-source SIEM solutions such as **Wazuh, Graylog & OSSIM** that can also be used depending on your environment and requirements.

- First attach Sentinel to your existing LAW.
- Then add the **Syslog (via AMA)** data connector and link it to your DCR.

![dataconnector](/assets/images/dc.png)

### Workbooks
- Workbooks allow you to visualise multiple datasets in a single, real-time dashboard
- Create a workbook in Sentinel → Workbooks → Create new.
 
![wb](/assets/images/wb.png)

- My workbook contains **3** key queries:

![wbqueries](/assets/images/wbqueries.png)


1. Aggregate data count (heartbeat, Syslog, JSON, Alerts etc.)

![query1](/assets/images/query1.png)


2. Syslog data - default but with cleaned columns.
> Note: In KQL `project-away` gets rid of chosen columns which makes data more clean and gets rid of blank/unnecessary columns.

![query2](/assets/images/query2.png)


3. Cowrie JSON logs - extracts attacker IPs, resolves geolocation via latitude/longitude and maps the source country & city.

![query3](/assets/images/query3.png)


### Creating automated alerts
- Sentinel can automatically trigger alerts when certain KQL conditions are met.

![alerts](/assets/images/alerts.png)

-  I kept it simple and created 2 alerts:
1. Brute Force SSH attempts - Detects repeated failed SSH logins from the same IP and creates an alert if it exceeds 2 attempts.

![bf](/assets/images/bfgeneral.png)
![bf1](/assets/images/bfrl1.png)
![bf2](/assets/images/bfrl2.png)

2. Successful SSH Logins - Detects successful logins and extracts the source IP for correlation.

![sl1](/assets/images/sl1.png)
![sl2](/assets/images/sl2.png)

## Automation
- We can also add some slight automation (nothing too crazy) by adding an automation rule onto our alerts
- When an incident is created the rule will automatically assign the incident to a specific person and give a task to that person.

![automation](/assets/images/automation.jpg)


