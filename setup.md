# 🧩 Setup & Configuration

#### ⚠️ Disclaimer: 
This is more of a report rather then a guide/walkthrough. I haven't documented every single step or command - I encourage readers to research, experiment, and adapt the setup to their own goals. Use this repository as an idea/starting point, not a step-by-step script.

## Project Goals
- Deploy a **Azure VM** as a secure, monitored host environment.
- Install & configure a Cowrie SSH honeypot to simulate an interactive **SSH** service.
- Forward system & honeypot logs to **Microsoft Sentinel** for detection & analysis.
- Build **KQL** queries, workbooks and alert rules for real-time attack detection & visualisation.

---

### Creating an Azure VM
- To host the honeypot, a **Virtual Machine (VM)** is required to act as the server environment.
- There are 2 main approaches to this: hosting a VM locally using virtualisation software **(VirtualBox, VMware)** or delpoying one through a **cloud provider** **(Azure, AWS, GCP)**
- Each method has its pros & cons, but I chose **Microsoft Azure** due to its **30-day free trial**, **generous credit allowance** and native integration with Microsoft Sentinel
> **Note:** AWS and GCP also give free trials
- Using cloud infrastructure also reflects real-world practices, giving hands-on experience in cloud security, resource management and security monitoring.

VM Configs:
> **Note:** All other sections were left as default

![vmb1](/images/vmbasics1.png)

> **Note:** You can also use **SSH public-key authentication** however i used password authentication for simplification and allow for brute-force attack simulation.

![vmb2](/images/vmbasics2.png)
![vmd](/images/vmdisk.png)

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

![nsg](/images/nsg.png)

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

![datasources](/images/ds.png)
![syslog](/images/syslog.png)
![json](/images/custjson.png)

#### Verification
- Test data ingestion in the LAW using `Syslog | take 10`
- If logs appear, the setup works.
- To ingest Cowrie JSON logs, first create a custom table in the LAW and define its schema using a sample JSON entry from `cowrie.json`.
- To test JSON logs use the same command as above but replace `Syslog` with your table name e.g `JSON_CL | take 10`
> **Note:** tables in LAW end in _CL

---

### Microsoft Sentinel
- This will allow us to collate logs into one area and set up alerts.
- First create a Sentinel subscription and create a resource which you can link to your existing LAW.
- Then create a data connector called Syslog via AMA and link that to the DCR.
- Now all logs can also come up on Sentinel.
- It should like like as shown under

![dataconnector](/images/dc.png)

### Workbooks
- a Workbook allows for multiple data points to come up on one page which is updated realtime.
- Creating a workbook is simple just click create a workbook on the Sentinel workbook tab.
 
![wb](/images/wb.png)

- I have 3 queries open on this tab; Syslog, JSON logs and an overall data collection query which shows amount of different logs (heartbeat, syslog, json) in a bar chart.

![wbqueries](/images/wbqueries.png)

- The first query is a default one that comes with all workbooks

![query1](/images/query1.png)

- The second query is a default Syslog query with KQL but with added code to remove unneccessary/blank
columns.

![query2](/images/query2.png)

- The third query is a JSON query with KQL but it adds geolocation which takes the location of the incoming ssh traffic by ip and gives it a longitutude/latitude and displays the potential country and city the attacker is in.

![query3](/images/query3.png)

### Creating automated alerts
- Sentinel allows us to create automated alerts if our KQL code is triggered.

![alerts](/images/alerts.png)

- We will create 2 alerts: successfull ssh login and bruteforce ssh attempts.
- Below you will see the configurations and KQL code for each of the alerts I have created.

![bf](/images/bfgeneral.png)

![bf1](/images/bfrl1.png)

- This alert looks for failed ssh attempts and if they surpass 2 the ip is extracted from the logs and an alert is made with the suspicous ip.

![bf2](/images/bfrl2.png)

![sl1](/images/sl1.png)
![sl2](/images/sl2.png)

- This does the same thing as the alert above but looks for successful logins and extracts the ip associated with that login.

