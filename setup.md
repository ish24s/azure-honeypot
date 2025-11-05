# 🧩 Setup & Configuration

## Project Goals
- Deploy a **Azure VM**.
- Configure the VM to run a **Cowrie SSH honeypot**.
- Forward honeypot logs to **Microsoft Sentinel** for detection & analysis.
- Build KQL queries & workbooks to visualise attack activity

---

### Creating an Azure VM
- To host the honeypot, a **Virtual Machine (VM)** is required to act as the server environment.
- There are 2 main approaches to this: hosting a VM locally using virtualisation software **(VirtualBox, VMware)** or delpoying one through a **cloud provider** **(Azure, AWS, GCP)**
- Each method has its own advantages and disadvantages, but I chose **Microsoft Azure** due to its **30-day free trial** & generous **credit allowance**. Furthemore it offers seamless compatibility with **Microsoft Sentinel** (more on that later), allowing for straightforward integration and log forwarding.
> Note: AWS and GCP also give free trials
- Since cloud platforms are widely used in modern times, gaining hands-on experience with them provides valuable **practical exposure** to modern infrastructure management and deployment practices.

VM Configs:
> Note: All other sections were left as default.

![vmb1](/images/vmbasics1.png)
> Note: You can also choose SSH publickey as the authentication type, I used password as it is more convenient and it can also be used to simulate brute force ssh attacks.

![vmb2](/images/vmbasics2.png)
![vmd](/images/vmdisk.png)

---

### Installing Cowrie
- To start with you will need a honeypot environment - common choices include **Cowrie**, **T-Pot**, etc.
- Cowrie simulates a fake SSH service which im familiar with therefore i chose Cowrie.
- I followed the official installation guide: https://docs.cowrie.org/en/latest/INSTALL.html.  
- My `cowrie.cfg` is included in this repo for reference.

### Configuring SSH on port 22
- By default Cowrie runs on port 2222 but it can listen on the standard SSH port (22) to appear realistic. To free port 22 for Cowrie, move your real SSH daemon to an alternate port.  
- Edit the system SSH config (`/etc/ssh/sshd_config`) and change the port e.g `Port 1111`
- Restart the SSH service after changing port (`sudo systemctl restart sshd`)
- Enable SSH in cowrie.cfg (already present in my config):

```ini
[ssh]
enabled = true
listen_endpoints = tcp:22:interface=0.0.0.0
```

- Now the last thing to do is edit the **NSG (Network Security Group)** on the Azure VM to allow inbound traffic for both ports: the honeypot (22) and your real SSH port (e.g 1111) 
![nsg](/images/nsg.png)

> Note: Give the rule for the real SSH port a higher priority (lower priority number) so you can still connect to the VM for administration. You should have two separate inbound rules: one for the real SSH port and one for the honeypot (22).

---

### Configuring a fake filesystem
- A fake filesystem makes the environment look real and provides interactive bait for attackers + it is fully customisable.
- First create a directory with your fake filesystem contents e.g
`sudo mkdir -p /opt/cowrie/custom_honeyfs/{etc,home,root,var/log,usr/bin,bin}`
- Generate the filesystem pickle (adjust paths to your installation `bin/createfs -l /home/cowrie/cowrie/customhoneyfs/ -d 3 -o customfs.pickle`
- Lastly point Cowrie to the generated file system in cowrie.cfg
```ini
[shell]
filesystem = /home/cowrie/cowrie/var/lib/cowrie/customfs.pickle
```
> Note: Paths in your installation may differ so use the correct paths for your VM 
- Add fake users and passwords to `/cowrie/etc/userdb.txt` to simulate realistic brute-force targets

---

### Log Forwarding
- We are going to forward 2 types of logs, **(Syslog and JSON)**. **Syslog** will give overall system logs as well as ssh logs and **JSON** will give Cowrie specific logs.
- We will need to edit cowrie.cfg to output JSON & Syslog logs.

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
- We also need a **Log Analytics Workspace (LAW)** to query all our logs.
- The config is fairly simple and all you need to do is link it to your previously created resource group and the same region your VM is in.
- Next we will need to add the **Azure Monitor Agent** to our VM to allow logs to be collected.
- Then configure both a **Data Collection Rule** and a **Data Collection Endpoint** to **Azure Monitor** to allow data collection.
- For the DCR, add both **Linux Syslog** as well as **Custom JSON logs**
- The **JSON schema** follows the same schema as the file **cowrie.json** in the cowrie log folder.
- The **Syslog** customisation is customisable, LOG_AUTH and LOG_AUTHPRIV are related to authentication logs ??? 
![datasources](/images/ds.png)
![syslog](/images/syslog.png)
![json](/images/custjson.png)

- To test whether logs are coming, go to the **LAW** and go on KQL mode and input `Syslog | take 10`
- If you see logs then it is working.
- To see JSON logs you need to first create a table with the same JSON schema. to do this create a table in the LAW and input a sample JSON line from the cowrie.json log file and edit to suit your goals.

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
- I have 3 queries open on this tab; Syslog, JSON logs and an overall data collection query which shows amount of different logs (heartbeat, syslog, json) in a bar chart.
- 
