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

### Installing and Configuring Cowrie
- To start with you will need a honeypot environment, there are various choices such as Cowrie, T-Pot etc.
- Cowrie simulates a fake ssh login which im familiar with therefore i chose cowrie.
- I wont give all the commands to download cowrie but i downloaded cowrie using the official install documentation on this link (https://docs.cowrie.org/en/latest/INSTALL.html)

