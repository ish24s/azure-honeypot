#  SSH Honeypot & SIEM Integration

## Documentation
- [Setup & Configuration](setup.md)
- [Analysis & Findings](analysis.md)
- [Simulation](simulation.md)

## 🎯 Project Overview

This project involves deploying a **Cowrie SSH Honeypot** in **Microsoft Azure** to attract and log malicious login attempts. The collected data is forwarded to **Microsoft Sentinel** for real-time analysis, correlation, and visualization using custom KQL queries and workbooks, offering practical experience in **threat detection, log monitoring & attack analysis** within a controlled, simulated environment.

The main objectives are:
- Deploy and configure a **Cowrie SSH honeypot** on an **Azure VM**.
- Forward **Syslog** and **JSON logs** to a **Log Analytics Workspace (LAW)**.
- Integrate with **Microsoft Sentinel** for event monitoring and alert creation.
- Develop **KQL queries** and **interactive dashboards** to analyse attack trends.

> ⚠️ Disclaimer: This is more of a **report** rather then a guide/walkthrough. I haven't documented every single step or command - I encourage readers to research, experiment, and adapt the setup to their own goals. Use this repository as an **idea/starting point**, not a **step-by-step script**.

---
## ⚙️ Key Components

| Component | Description |
|------------|--------------|
| **Azure VM** | Cloud-hosted Linux server for running the honeypot |
| **Cowrie Honeypot** | Simulated SSH environment with many features such as log collection & session recording |
| **Log Analytics Workspace** | Collects and queries system & honeypot logs |
| **Microsoft Sentinel** | SIEM platform for correlation, alerting & dashboards |
| **KQL Queries** | Used for data filtering, IP extraction & alert logic |
| **Data Collection Rules** | Defines how and what type of data is collected |


---

## 📊 Features

- SSH honeypot configured to run on **port 22** with realistic file system emulation & terminal recording.
- Dual log forwarding setup (**Syslog + JSON**) via **Azure Monitor Agent (AMA)**.
- **Sentinel Workbooks** visualising various data fields including source IP, geolocation & log type
- Automated alerts detecting **brute-force attempts** and **successful logins**.
- Clean, documented configuration for **educational & practical reference**.

---

## 🔗 Useful Links
- [Cowrie Documentation](https://docs.cowrie.org/en/latest/)
- [Microsoft Azure VM Documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/)
- [Microsoft Sentinel Overview](https://learn.microsoft.com/en-us/azure/sentinel/)
- [KQL Query Language Guide](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/)
- [Alternative Honeypots](https://github.com/paralax/awesome-honeypots)
- [Alternative SIEM's](https://redcanary.com/cybersecurity-101/security-operations/top-free-siem-tools/)
